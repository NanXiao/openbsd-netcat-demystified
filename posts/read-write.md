# Read & Write

Once the connection is established, it is time to transfer data. The `readwrite` is the core function of `netcat`, and thankfully, it is well-commented, and refrain you from losing. In this post, I will only cherry-pick code snippets that I think is important.

(1) There is a `-d` option that if set, your `netcat` program can not send data inputted from `stdin`:  

	......
	int	dflag;					/* detached, no stdin */

	/* don't read from stdin if requested */
	if (dflag)
		stdin_fd = -1;


(2) `readwrite` uses [poll](https://man.openbsd.org/poll.2) to manage `3` file descriptors: `stdin`, `stdout`, and network socket, and monitor whether there is data from `stdin` or network socket. If there is no need for “high-priority” (e.g., out-of-band) data, `POLLIN` is enough, otherwise the monitor events should be `POLLIN`|`POLLPRI`. This is similar for `POLLOUT` and `POLLWRBAND`:  

	......
	/* stdin */
	pfd[POLL_STDIN].fd = stdin_fd;
	pfd[POLL_STDIN].events = POLLIN;

	/* network out */
	pfd[POLL_NETOUT].fd = net_fd;
	pfd[POLL_NETOUT].events = 0;

	/* network in */
	pfd[POLL_NETIN].fd = net_fd;
	pfd[POLL_NETIN].events = POLLIN;

	/* stdout */
	pfd[POLL_STDOUT].fd = stdout_fd;
	pfd[POLL_STDOUT].events = 0;
	......

	/* poll */
	num_fds = poll(pfd, 4, timeout);

	/* treat poll errors */
	if (num_fds == -1)
		err(1, "polling error");

	/* timeout happened */
	if (num_fds == 0)
		return;

There are 3 values(`POLLERR`, `POLLNVAL` and `POLLHUP`) which are only used in `struct pollfd`‘s `revents`. If `POLLERR` or `POLLNVAL` is detected, it’s not necessary to poll this file descriptor furthermore:  

	/* treat socket error conditions */
	for (n = 0; n < 4; n++) {
		if (pfd[n].revents & (POLLERR|POLLNVAL)) {
			pfd[n].fd = -1;
		}
	}
`POLLHUP` means the device or network socket has been disconnected, and it is  exclusive with `POLLOUT`, but not input events:`POLLIN`, `POLLRDNORM`, `POLLRDBAND`, and `POLLPRI`. That's because the buffer may still has data to be read:  

	/* reading is possible after HUP */
	if (pfd[POLL_STDIN].events & POLLIN &&
	    pfd[POLL_STDIN].revents & POLLHUP &&
	    !(pfd[POLL_STDIN].revents & POLLIN))
			pfd[POLL_STDIN].fd = -1;
	......
	if (pfd[POLL_NETOUT].revents & POLLHUP) {
		if (Nflag)
			shutdown(pfd[POLL_NETOUT].fd, SHUT_WR);
		pfd[POLL_NETOUT].fd = -1;
	}
BTW, the `Nflag` is set through `-N` option, which means shutting down the network socket if there is no any input or the network connection is already disconnected:  

	int	Nflag;					/* shutdown() network socket */
	......
	/* stdin gone and queue empty? */
	if (pfd[POLL_STDIN].fd == -1 && stdinbufpos == 0) {
		if (pfd[POLL_NETOUT].fd != -1 && Nflag)
			shutdown(pfd[POLL_NETOUT].fd, SHUT_WR);
		pfd[POLL_NETOUT].fd = -1;
	}

(3) The following code shows how to read from `stdin` and send data to remote:

	......
	#define BUFSIZE		16384
	.....
	unsigned char stdinbuf[BUFSIZE];
	size_t stdinbufpos = 0;

	......

	/* try to read from stdin */
	if (pfd[POLL_STDIN].revents & POLLIN && stdinbufpos < BUFSIZE) {
		ret = fillbuf(pfd[POLL_STDIN].fd, stdinbuf,
			 &stdinbufpos, NULL);
		if (ret == TLS_WANT_POLLIN)
			pfd[POLL_STDIN].events = POLLIN;
		else if (ret == TLS_WANT_POLLOUT)
			pfd[POLL_STDIN].events = POLLOUT;
		else if (ret == 0 || ret == -1)
			pfd[POLL_STDIN].fd = -1;
		/* read something - poll net out */
		if (stdinbufpos > 0)
			pfd[POLL_NETOUT].events = POLLOUT;
		/* filled buffer - remove self from polling */
		if (stdinbufpos == BUFSIZE)
			pfd[POLL_STDIN].events = 0;
	}
	/* try to write to network */
	if (pfd[POLL_NETOUT].revents & POLLOUT && stdinbufpos > 0) {
		ret = drainbuf(pfd[POLL_NETOUT].fd, stdinbuf,
			 &stdinbufpos, tls_ctx);
		if (ret == TLS_WANT_POLLIN)
			pfd[POLL_NETOUT].events = POLLIN;
		else if (ret == TLS_WANT_POLLOUT)
			pfd[POLL_NETOUT].events = POLLOUT;
		else if (ret == -1)
			pfd[POLL_NETOUT].fd = -1;
		/* buffer empty - remove self from polling */
		if (stdinbufpos == 0)
			pfd[POLL_NETOUT].events = 0;
		/* buffer no longer full - poll stdin again */
		if (stdinbufpos < BUFSIZE)
			pfd[POLL_STDIN].events = POLLIN;
	}
Please ignore all `TLS*` stuff now. It just means using `libtls` instead of traditional `read/write` APIs to receive/send data, and I will cover `libtls` in future. `stdinbufpos` denotes the first empty position in buffer, therefore, `netcat` knows how many bytes could be sent.  

`fillbuf` and `drainbuf` code is like following:

	ssize_t
	fillbuf(int fd, unsigned char *buf, size_t *bufpos, struct tls *tls)
	{
		size_t num = BUFSIZE - *bufpos;
		ssize_t n;
	
		if (tls)
			n = tls_read(tls, buf + *bufpos, num);
		else {
			n = read(fd, buf + *bufpos, num);
			/* don't treat EAGAIN, EINTR as error */
			if (n == -1 && (errno == EAGAIN || errno == EINTR))
				n = TLS_WANT_POLLIN;
		}
		if (n <= 0)
			return n;
		*bufpos += n;
		return n;
	}

	ssize_t
	drainbuf(int fd, unsigned char *buf, size_t *bufpos, struct tls *tls)
	{
		ssize_t n;
		ssize_t adjust;
	
		if (tls)
			n = tls_write(tls, buf, *bufpos);
		else {
			n = write(fd, buf, *bufpos);
			/* don't treat EAGAIN, EINTR as error */
			if (n == -1 && (errno == EAGAIN || errno == EINTR))
				n = TLS_WANT_POLLOUT;
		}
		if (n <= 0)
			return n;
		/* adjust buffer */
		adjust = *bufpos - n;
		if (adjust > 0)
			memmove(buf, buf + n, adjust);
		*bufpos -= n;
		return n;
	}
You should get `2` takeaways from preceding code:  
a) `EAGAIN` and `EINTR` are not considered as real errors;  
b) Don't forget to adjust `bufpos` and `buf`, i.e., `stdinbufpos` and `stdinbuf` accordingly.  

As for reading data from network socket and outputting it to `stdout` flow, it is very similar, so there is no need to say more here.  

(4) There is a `-W recvlimit` option which specifies after receiving how many packets, the program should terminate:  

	......
	int recvcount, recvlimit;
	......
	/* try to read from network */
	if (pfd[POLL_NETIN].revents & POLLIN && netinbufpos < BUFSIZE) {
		......
		if (recvlimit > 0 && ++recvcount >= recvlimit) {
			if (pfd[POLL_NETIN].fd != -1)
				shutdown(pfd[POLL_NETIN].fd, SHUT_RD);
			pfd[POLL_NETIN].fd = -1;
			pfd[POLL_STDIN].fd = -1;
		}
		......
	}