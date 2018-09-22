# Connect server
Other than server's socket, client's socket works in non-block mode:  

	if ((s = socket(res->ai_family, res->ai_socktype |
		    SOCK_NONBLOCK, res->ai_protocol)) < 0)
`timeout_connect()` function is used by client to connect server:  

	int
	timeout_connect(int s, const struct sockaddr *name, socklen_t namelen)
	{
		struct pollfd pfd;
		socklen_t optlen;
		int optval;
		int ret;
	
		if ((ret = connect(s, name, namelen)) != 0 && errno == EINPROGRESS) {
			pfd.fd = s;
			pfd.events = POLLOUT;
			if ((ret = poll(&pfd, 1, timeout)) == 1) {
				optlen = sizeof(optval);
				if ((ret = getsockopt(s, SOL_SOCKET, SO_ERROR,
				    &optval, &optlen)) == 0) {
					errno = optval;
					ret = optval == 0 ? 0 : -1;
				}
			} else if (ret == 0) {
				errno = ETIMEDOUT;
				ret = -1;
			} else
				err(1, "poll failed");
		}
	
		return ret;
	}

We will analyze it step by step:  

(1) There is a `timeout` variable that can be specified by `-w` option (default value is `-1`, means no timeout), and denotes how long [connect()](https://man.openbsd.org/connect.2) function can wait (the unit is seconds):  

	int timeout = -1;
	......
	while ((ch = getopt(argc, argv, ...))) {
		    ......
			switch (ch) {
			......
			case 'w':
				timeout = strtonum(optarg, 0, INT_MAX / 1000, &errstr);
				if (errstr)
					errx(1, "timeout %s: %s", errstr, optarg);
				timeout *= 1000;
				break;
			......
			}
	}

(2) Check [connect()](https://man.openbsd.org/connect.2)'s return value: if return value is `0`, it means connection is established successfully; if `errno` is `EINPROGRESS`, we can use `timeout` to control how long to wait; otherwise the connection can’t be built:   

	if ((ret = connect(s, name, namelen)) != 0 && errno == EINPROGRESS) {
	......
	}
(3) Use [poll()](https://man.openbsd.org/poll.2) to monitor whether the connection is successful or not. The `POLLOUT` event means socket become writable. Even return value of `poll` is `1`, the [getsockopt()](https://man.openbsd.org/getsockopt.2) can be used to check whether there is indeed no error.  

P.S., generally speaking, because `UDP` is connection-less protocol, there is no need to call [connect()](https://man.openbsd.org/connect.2) for `UDP` client.