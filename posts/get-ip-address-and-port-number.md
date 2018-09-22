# Get IP address and Port number
No matter `netcat` works in client or server mode, using `TCP` or `UDP`, getting the `IP` address and port number is always the first step (`UNIX-domain` socket doesn't need it definitely). By default, `netcat` will work in `TCP` mode, except you specify `-u` option. There is a variable which denotes using `UDP` or not:  

	int	uflag;					/* UDP - Default to TCP */
	......
	while ((ch = getopt(argc, argv, ...))) {
		    ......
			switch (ch) {
			......
			case 'u':
				uflag = 1;
				break;
			......
			}
	}
[getaddrinfo()](https://man.openbsd.org/getaddrinfo.3) fulfills the duty to get `IP` address and port number:  

     int
     getaddrinfo(const char *hostname, const char *servname,
         const struct addrinfo *hints, struct addrinfo **res);

The third parameter's type is `struct addrinfo`, which gives a hint of what kind of socket the application wants to use:

	struct addrinfo hints;
	......
	/* Initialize addrinfo structure. */
	if (family != AF_UNIX) {
		memset(&hints, 0, sizeof(struct addrinfo));
		hints.ai_family = family;
		hints.ai_socktype = uflag ? SOCK_DGRAM : SOCK_STREAM;
		hints.ai_protocol = uflag ? IPPROTO_UDP : IPPROTO_TCP;
		if (nflag)
			hints.ai_flags |= AI_NUMERICHOST;
	}
 The only thing that needs to pay attention is a `nflag`, which will be set through `-n` option. It means don't bother to do DNS query or service look-up, the `IP` address and port number are already provided.  

Because server's socket will be used in future [bind()](https://man.openbsd.org/bind.2) operation, it has another extra step than client: 

	/* Allow nodename to be null. */
	hints.ai_flags |= AI_PASSIVE;
This means `INADDR_ANY` (For `IPv4`) or `IN6ADDR_ANY_INIT`  (For `IPv4`) will be used if `hostname` parameter for [getaddrinfo()](https://man.openbsd.org/getaddrinfo.3) is `NULL`.   

Now that all is ready, it is time to get `IP` address, port number and create socket accordingly:  

	if ((error = getaddrinfo(host, port, &hints, &res0)))
		errx(1, "getaddrinfo: %s", gai_strerror(error));

	for (res = res0; res; res = res->ai_next) {
		if ((s = socket(res->ai_family, res->ai_socktype,
		    res->ai_protocol)) < 0)
			continue;
		......
	}

Last but not least, don't forget to release the memory of querying result:  

	freeaddrinfo(res0);

Now you can understand how `localhost` and `www` is mapped to `IP` address and `port` number in the following commands:  

	# nc -l localhost www
	# nc localhost www
 


 