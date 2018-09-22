# Some socket options

In fact, both network socket client & server will call a `set_common_sockopts` function which will set some socket options according to command line options:  

	void
	set_common_sockopts(int s, int af)
	{
		int x = 1;
	
		if (Sflag) {
			if (setsockopt(s, IPPROTO_TCP, TCP_MD5SIG,
				&x, sizeof(x)) == -1)
				err(1, NULL);
		}
		if (Dflag) {
			if (setsockopt(s, SOL_SOCKET, SO_DEBUG,
				&x, sizeof(x)) == -1)
				err(1, NULL);
		}
		if (Tflag != -1) {
			if (af == AF_INET && setsockopt(s, IPPROTO_IP,
			    IP_TOS, &Tflag, sizeof(Tflag)) == -1)
				err(1, "set IP ToS");
	
			else if (af == AF_INET6 && setsockopt(s, IPPROTO_IPV6,
			    IPV6_TCLASS, &Tflag, sizeof(Tflag)) == -1)
				err(1, "set IPv6 traffic class");
		}
		if (Iflag) {
			if (setsockopt(s, SOL_SOCKET, SO_RCVBUF,
			    &Iflag, sizeof(Iflag)) == -1)
				err(1, "set TCP receive buffer size");
		}
		if (Oflag) {
			if (setsockopt(s, SOL_SOCKET, SO_SNDBUF,
			    &Oflag, sizeof(Oflag)) == -1)
				err(1, "set TCP send buffer size");
		}
	
		if (ttl != -1) {
			if (af == AF_INET && setsockopt(s, IPPROTO_IP,
			    IP_TTL, &ttl, sizeof(ttl)))
				err(1, "set IP TTL");
	
			else if (af == AF_INET6 && setsockopt(s, IPPROTO_IPV6,
			    IPV6_UNICAST_HOPS, &ttl, sizeof(ttl)))
				err(1, "set IPv6 unicast hops");
		}
	
		if (minttl != -1) {
			if (af == AF_INET && setsockopt(s, IPPROTO_IP,
			    IP_MINTTL, &minttl, sizeof(minttl)))
				err(1, "set IP min TTL");
	
			else if (af == AF_INET6 && setsockopt(s, IPPROTO_IPV6,
			    IPV6_MINHOPCOUNT, &minttl, sizeof(minttl)))
				err(1, "set IPv6 min hop count");
		}
	}
Below is the table which summarize these options:  

<table>
  <tr><td>Variable</td><td>Command option</td><td>Socket level & option</td><td>Meaning</td></tr>
  <tr><td>Sflag</td><td>-S</td><td>IPPROTO_TCP&TCP_MD5SIG</td><td>Uses TCP MD5 signatures per RFC 2385</td></tr>
  <tr><td>Dflag</td><td>-D</td><td>SOL_SOCKET&SO_DEBUG</td><td>Enables recording of debugging information</td></tr>
  <tr><td>Tflag</td><td>-T</td><td>IPPROTO_IP&IP_TOS IPPROTO_IPV6&IPV6_TCLASS</td><td>Changes IPv4 TOS value (refer process_tos_opt function) or sets IPv6 traffic class field</td></tr>
  <tr><td>Iflag</td><td>-I length</td><td>SOL_SOCKET&SO_RCVBUF</td><td>Sets buffer size for input</td></tr>
  <tr><td>Oflag</td><td> -O length</td><td>SOL_SOCKET&SO_SNDBUF</td><td>Sets buffer size for output</td></tr>
  <tr><td>ttl</td><td> -M ttl</td><td>IPPROTO_IP&IP_TTL IPPROTO_IPV6&IPV6_UNICAST_HOPS</td><td>Sets the TTL / hop limit of outgoing packets</td></tr>
  <tr><td>minttl</td><td> -m minttl</td><td>IPPROTO_IP&IP_MINTTL IPPROTO_IPV6&IPV6_MINHOPCOUNT</td><td>Discards packets with a TTL lower than the option value</td></tr>
</table>


