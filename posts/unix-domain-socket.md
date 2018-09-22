# Unix-domain socket

`Unix-domain` socket code is akin to its network counterpart, and they share one `readwrite()` function. So I only list the caveats during creating `Unix-domain` socket connection:  

(1) Like network socket, `Unix-domain` socket also supports connection & connection-less services; instead of using `IP` address + port, `Unix-domain` should bind to a filesystem pathname. Check `unix_bind` function:  

	/*
	 * unix_bind()
	 * Returns a unix socket bound to the given path
	 */
	int
	unix_bind(char *path, int flags)
	{
		struct sockaddr_un s_un;
		int s, save_errno;
	
		/* Create unix domain socket. */
		if ((s = socket(AF_UNIX, flags | (uflag ? SOCK_DGRAM : SOCK_STREAM),
		    0)) < 0)
			return -1;
	
		memset(&s_un, 0, sizeof(struct sockaddr_un));
		s_un.sun_family = AF_UNIX;
	
		if (strlcpy(s_un.sun_path, path, sizeof(s_un.sun_path)) >=
		    sizeof(s_un.sun_path)) {
			close(s);
			errno = ENAMETOOLONG;
			return -1;
		}
	
		if (bind(s, (struct sockaddr *)&s_un, sizeof(s_un)) < 0) {
			save_errno = errno;
			close(s);
			errno = save_errno;
			return -1;
		}
	
		return s;
	}

(2) The biggest surprise is connection-less `Unix-domain` client also needs to be bound to a pathname:  

	......
	/* Get name of temporary socket for unix datagram client */
	if ((family == AF_UNIX) && uflag && !lflag) {
		if (sflag) {
			unix_dg_tmp_socket = sflag;
		} else {
			strlcpy(unix_dg_tmp_socket_buf, "/tmp/nc.XXXXXXXXXX",
			    UNIX_DG_TMP_SOCKET_SIZE);
			if (mktemp(unix_dg_tmp_socket_buf) == NULL)
				err(1, "mktemp");
			unix_dg_tmp_socket = unix_dg_tmp_socket_buf;
		}
	}
	......
`sflag` will be set with `-s source` option:  

	......
	char   *sflag;					/* Source Address */
	......
So if user doesn't specify a pathname for client, the client will create a temporary socket.  

Therefore, the connection-less client also needs to call `unix_bind()` before connecting server:  

	/*
	 * unix_connect()
	 * Returns a socket connected to a local unix socket. Returns -1 on failure.
	 */
	int
	unix_connect(char *path)
	{
		struct sockaddr_un s_un;
		int s, save_errno;
	
		if (uflag) {
			if ((s = unix_bind(unix_dg_tmp_socket, SOCK_CLOEXEC)) < 0)
				return -1;
		} else {
			if ((s = socket(AF_UNIX, SOCK_STREAM | SOCK_CLOEXEC, 0)) < 0)
				return -1;
		}
	
		......
		if (connect(s, (struct sockaddr *)&s_un, sizeof(s_un)) < 0) {
			......
		}
		return s;
	
	}

For the connection-less server part, there is also one more step needed:  

	......
	if (family == AF_UNIX && uflag) {
		if (connect(s, NULL, 0) < 0)
			err(1, "connect");
	}
	......






