# Port scanning

`netcat` can be used as "hacking" tool, e.g., `-z` option for port scanning:  

	# nc -vz google.com 443-445
	Connection to google.com 443 port [tcp/https] succeeded!
	nc: connect to google.com port 444 (tcp) failed: Operation timed out
	nc: connect to google.com port 445 (tcp) failed: Operation timed out
First, `build_ports` parses `443-445` to generate port lists:  

	void
	build_ports(char *p)
	{
		char *n;
		int hi, lo, cp;
		int x = 0;
	
		if ((n = strchr(p, '-')) != NULL) {
			*n = '\0';
			n++;
	
			/* Make sure the ports are in order: lowest->highest. */
			hi = strtoport(n, uflag);
			lo = strtoport(p, uflag);
			if (lo > hi) {
				cp = hi;
				hi = lo;
				lo = cp;
			}
	
			/*
			 * Initialize portlist with a random permutation.  Based on
			 * Knuth, as in ip_randomid() in sys/netinet/ip_id.c.
			 */
			if (rflag) {
				for (x = 0; x <= hi - lo; x++) {
					cp = arc4random_uniform(x + 1);
					portlist[x] = portlist[cp];
					if (asprintf(&portlist[cp], "%d", x + lo) < 0)
						err(1, "asprintf");
				}
			} else { /* Load ports sequentially. */
				for (cp = lo; cp <= hi; cp++) {
					if (asprintf(&portlist[x], "%d", cp) < 0)
						err(1, "asprintf");
					x++;
				}
			}
		} else {
			......
		}
	}

The `portlist` array is defined as following:  

	......
	#define PORT_MAX	65535
	......
	char *portlist[PORT_MAX+1];
	......

One feature of is generating port lists randomly, not in sequence through `-r` option:  

	......
	if (rflag) {
		for (x = 0; x <= hi - lo; x++) {
			cp = arc4random_uniform(x + 1);
			portlist[x] = portlist[cp];
			if (asprintf(&portlist[cp], "%d", x + lo) < 0)
				err(1, "asprintf");
		}
	}
	......
The interesting part is [arc4random_uniform()](https://man.openbsd.org/arc4random_uniform.3), which guarantees `cp` is less than or equal `x`.  So if `cp == x`:  

	......
	portlist[x] = portlist[cp];
	if (asprintf(&portlist[cp], "%d", x + lo) < 0)
		err(1, "asprintf");
	......
`portlist[x]` and `portlist[cp]` point to same slot. Otherwise, if `cp < x`, put the original value of `portlist[cp]` to `portlist[x]`, and generate a new value of `portlist[cp]`.  

After generating port list, try connecting the port to check whether it is open or not. For `UDP` service, there is a further step to test:  

	/*
	 * udptest()
	 * Do a few writes to see if the UDP port is there.
	 * Fails once PF state table is full.
	 */
	int
	udptest(int s)
	{
		int i, ret;
	
		for (i = 0; i <= 3; i++) {
			if (write(s, "X", 1) == 1)
				ret = 1;
			else
				ret = -1;
		}
		return ret;
	}

Finally, [getservbyport()](https://man.openbsd.org/getservbyport.3) is used to display service name:   

	......
	/* Don't look up port if -n. */
	if (nflag)
		sv = NULL;
	else {
		sv = getservbyport(
		    ntohs(atoi(portlist[i])),
		    uflag ? "udp" : "tcp");
	}

	fprintf(stderr,
	    "Connection to %s %s port [%s/%s] "
	    "succeeded!\n", host, portlist[i],
	    uflag ? "udp" : "tcp",
	    sv ? sv->s_name : "*");
	......
BTW, `nflag` is set by `-n` option, means not do service look-up.


