# Getting Started with boot2root 

## Enumeration

Enumeration is normally the key to success for these kind of challenges

> Important information that one will need
>
> - IP (duh)
> - Port Open (TCP / UDP )
> - Services running behind port ( e.g. ssh can run behind port 2122 )



### Getting Started

```
nmap -sC -sV -sS -oA <filename> <ip>
```

nmap is quite a good tools to do basic enumeration.

```
-sC option for default script
-sV for banner grabbing ( Service behind port)
-oA to output result into 3 format ( grep-able / xml / default-format )
```

One should read up on more nmap options to do more powerful scanning. 

What I would normally do is use other scanning tool (such as `unicornscan` or `masscan`) to find opened port before feeding the information to nmap

```
e.g.

masscan -e <interface> -p 1-65535 <ip>

using the result 

nmap -p <found ports> -sV -sS -A -oA <output-filename> <ip>
```

This method will be much faster than scanning all 65535 ports using nmap.

#### Manual Enumeration with `nc`

If say you do not have access to nmap. (SSRF related *cough cough*). You can make use of nc to find open port.

```
nc -nvv -w 1 -z <ip> 1-65535

nc -nv -u -z -w 1 <port> (udp)
```

But to note that this is going to take VERY long. --  *tcp 3-way handshake* 



Once you found out what are the services available. 

You will then be able to chain information which eventually lead to a shell. 



## Privilege Escalation

A good starting point would be from the following URL

https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/

This usually require some experience. 



## Useful Tool / Command

- nc / ncat
- nmap / unicornscan / masscan
- searchsploit ( exploit-db.com )
- LinEnum 
- curl 
- Burp suite
- 
