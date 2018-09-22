# OpenBSD netcat demystified
Owing to its versatile functionalities, [netcat](https://en.wikipedia.org/wiki/Netcat) earns the reputation as "TCP/IP Swiss army knife". For example, you can create a simple chat app using `netcat`:  

(1) Open a terminal and input following command:  

	# nc -l 3003	
This means a `netcat` process will listen on `3003` port in this machine (the `IP` address of current machine is `192.168.35.176`).  

(2) Connect aforemontioned `netcat` process in another machine, and send a greeting:  

	# nc 192.168.35.176 3003
	hello
Then in the first machine's terminal, you will see the "hello" text:  

	# nc -l 3003
	hello
A primitive chatroom is built successfully. Very cool! Isn't it? I think many people can't wait to explore more features of `netcat`now.  If you are among them, congratulations! This tutorial may be the correct place for you. 

In the following parts, I will delve into `OpenBSD`'s `netcat`code to give a detailed anatomy of it. The reason of picking `OpenBSD`'s `netcat` rather than others' is because its code repository is small (`~2000` lines of code) and neat. Furthermore, I also hope this little book can assist you learn more socket programming knowledge not just grasping usage of `netcat`.

We're all set. Let's go!