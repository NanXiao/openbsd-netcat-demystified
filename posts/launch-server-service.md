# Launch server service

Once socket is created successfully, we can launch server or client. Let's focus on server firstly.  

No matter server uses `TCP` or `UDP`, the following code is shared (in `local_listen` function):  

	......
	ret = setsockopt(s, SOL_SOCKET, SO_REUSEPORT, &x, sizeof(x));
	if (ret == -1)
		err(1, NULL);

	set_common_sockopts(s, res->ai_family);

	if (bind(s, (struct sockaddr *)res->ai_addr,
		res->ai_addrlen) == 0)
		break; 
	......

`SO_REUSEPORT` is a common option when implementing server. About its detailed explanation, you can refer UNIX Network Programming, Volume 1, chapter 7, Socket Options. Personally, I think the `SO_REUSEPORT`'s most important  effect is you can bind port successfully after restarting server immediately. Otherwise you will encounter following annoying error:  

	Address already in use

The `set_common_sockopts()` will be discussed further. [bind()](https://man.openbsd.org/bind.2) function is used to bind socket and pair of `IP` address & port.  

If the server uses connection-less `UDP` protocol, then all work is done. But for `TCP`, there are `2` further steps:  

	......
	if (!uflag && s != -1) {
		if (listen(s, 1) < 0)
			err(1, "listen");
	}
	......

	socklen_t len;
	struct sockaddr_storage cliaddr;
	......
	len = sizeof(cliaddr);
	connfd = accept4(s, (struct sockaddr *)&cliaddr,
				    &len, SOCK_NONBLOCK);
	......

Server waits for the client's connection, and only serves only one connection at a time. Once the connection is established successfully, [accept4()](https://man.openbsd.org/accept4.2) will return new socket that is in `non-blocking` mode. Now the server can use the fresh socket to communicate with client.