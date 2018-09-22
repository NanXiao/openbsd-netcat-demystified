# SSL/TLS connection

`OpenBSD`'s `netcat` supports `SSL/TLS` connection, and uses `libtls` shipped by [libressl](https://www.libressl.org/) under the hood. The `-c` option is used to denote using `SSL/TLS`:  

	......
	int	usetls;					/* use TLS */
	......
	case 'c':
			usetls = 1;
			break;
There is a simple example which demonstrates how to leverage `netcat` as an `https` client:  

	# nc -c www.google.com https
	GET / HTTP/1.1
	Host: www.google.com
	Connection: close
	
	HTTP/1.1 200 OK
	Date: Fri, 21 Sep 2018 08:19:19 GMT
	Expires: -1
	Cache-Control: private, max-age=0
	Content-Type: text/html; charset=ISO-8859-1
	P3P: CP="This is not a P3P policy! See g.co/p3phelp for more info."
	Server: gws
	X-XSS-Protection: 1; mode=block
	X-Frame-Options: SAMEORIGIN
	......

The following is a sketch of how to use `libtls` to implement server and client:  
 
(1) Initialization:  
No matter `netcat` works as serve or client, the following code is common:  

	if (usetls) {
		if ((tls_cfg = tls_config_new()) == NULL)
			errx(1, "unable to allocate TLS config");
		if (Rflag && tls_config_set_ca_file(tls_cfg, Rflag) == -1)
			errx(1, "%s", tls_config_error(tls_cfg));
		if (Cflag && tls_config_set_cert_file(tls_cfg, Cflag) == -1)
			errx(1, "%s", tls_config_error(tls_cfg));
		if (Kflag && tls_config_set_key_file(tls_cfg, Kflag) == -1)
			errx(1, "%s", tls_config_error(tls_cfg));
		if (oflag && tls_config_set_ocsp_staple_file(tls_cfg, oflag) == -1)
			errx(1, "%s", tls_config_error(tls_cfg));
		if (tls_config_parse_protocols(&protocols, tls_protocols) == -1)
			errx(1, "invalid TLS protocols `%s'", tls_protocols);
		......
	}

[tls_config_new()](https://man.openbsd.org/tls_config_new.3) returns a new default configuration object. For other options and settings, I won't elaborate them here.   

Then, as `SSL/TLS` server, [tls_server()](https://man.openbsd.org/tls_server.3)  should be called and return a context object:

	......
	if ((tls_ctx = tls_server()) == NULL)
		errx(1, "tls server creation failed");
	......
Similarly, client invokes [tls_client()](https://man.openbsd.org/tls_client.3):  

	......
	if ((tls_ctx = tls_client()) == NULL)
		errx(1, "tls client creation failed");
	......
Both server and client should associate configuration and context objects:  

	if (tls_configure(tls_ctx, tls_cfg) == -1)
		errx(1, "tls configuration failed (%s)",
			tls_error(tls_ctx));

(2) Next step is associating exist socket with `SSL/TLS` context object:   
a) Server uses [tls_accept_socket()](https://man.openbsd.org/tls_accept_socket.3):  

	......
	if (tls_accept_socket(tls_ctx, &tls_cctx, connfd) == -1) {
		warnx("tls accept failed (%s)", tls_error(tls_ctx));
	}
	......
b) Client uses [tls_connect_socket()](https://man.openbsd.org/tls_connect_socket.3):  

	......
	if (tls_connect_socket(tls_ctx, s,
		tls_expectname ? tls_expectname : host) == -1) {
		errx(1, "tls connection failed (%s)",
		    tls_error(tls_ctx));
	}
	......

(3) Read & Write:   
After `SSL/TLS` connection is established, you can use [tls_read()](https://man.openbsd.org/tls_read.3) and [tls_write](https://man.openbsd.org/tls_write.3) to receive and send data. Different with [read()](https://man.openbsd.org/read.2) and [write()](https://man.openbsd.org/write.2) system calls, [tls_read()](https://man.openbsd.org/tls_read.3) and [tls_write](https://man.openbsd.org/tls_write.3) will process `2` more return values: [TLS_WANT_POLLIN](https://man.openbsd.org/tls_read.3#TLS_WANT_POLLIN) and [TLS_WANT_POLLOUT](https://man.openbsd.org/tls_read.3#TLS_WANT_POLLIN). E.g.:  

	......
	ret = fillbuf(pfd[POLL_STDIN].fd, stdinbuf,
			    &stdinbufpos, NULL);
	if (ret == TLS_WANT_POLLIN)
		pfd[POLL_STDIN].events = POLLIN;
	else if (ret == TLS_WANT_POLLOUT)
		pfd[POLL_STDIN].events = POLLOUT;
	......