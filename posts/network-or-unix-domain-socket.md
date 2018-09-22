# Network or UNIX-domain socket

`OpenBSD`'s `netcat` can support both network(based on `TCP/IP` or `UDP/IP`) and `UNIX-domain` sockets. `netcat` will use network sockets by default except you specify `-U` option. E.g, create and listen on a `UNIX-domain` socket:  

	nc -lU /var/tmp/unix-socket

`-l` option lets `netcat` work in server mode. There is a `lflag` which identifies `netcat` launched as server or client:

	int	lflag;					/* Bind to local port */


Regarding to network socket, `netcat` has `2` options to specify using `IPv4` only or `IPv6` only:  

     -4      Use IPv4 addresses only.
     -6      Use IPv6 addresses only.

`Netcat` program has a `family` variable which denotes using `IPv4`, `IPv6` or `UNIX-domain` socket:  

	int family = AF_UNSPEC;

During parsing options, `lflag` and `family` will be assigned different values accordingly:  

	while ((ch = getopt(argc, argv, ...))) {
		    ......
			switch (ch) {
			case '4':
				family = AF_INET;
				break;
			case '6':
				family = AF_INET6;
				break;
			case 'U':
				family = AF_UNIX;
				break;
			......
			case 'l':
				lflag = 1;
				break;
			......
			}
	}
`Netcat` will accept one or two arguments:  

	argc -= optind;
	argv += optind;
	......
	/* Cruft to make sure options are clean, and used properly. */
	if (argv[0] && !argv[1] && family == AF_UNIX) {
		host = argv[0];
		uport = NULL;
	} else if (argv[0] && !argv[1]) {
		if (!lflag)
			usage(1);
		uport = argv[0];
		host = NULL;
	} else if (argv[0] && argv[1]) {
		host = argv[0];
		uport = argv[1];
	} else
		usage(1);

If it is `UNIX-domain`, `netcat` only needs one argument which is the `UNIX-domain` socket file. If it is network socket, client must have two arguments: server `IP` address and port; server can omit `IP` address and only has port.

What `IP` protocol version will be used by `TCP/IP` server if neither `-4` nor `-6` is provided in option? The answer is  `IPv4`. In `local_listen` function:  

	/*
	 * In the case of binding to a wildcard address
	 * default to binding to an ipv4 address.
	 */
	if (host == NULL && hints.ai_family == AF_UNSPEC)
		hints.ai_family = AF_INET;
"`host == NULL`" means you don't designate an `IP` address for server, like following example:  

	# nc -l 3003