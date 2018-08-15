# How to Use an External Program to Handle Authentication instead of Apache

There are times when we have some kind of custom authentication system in place and apache provides auth modules for handling auth by checking credentials in a file or particular type of database and that locks if we want to integrate the custom auth system with apache.

But there is a module to hook a custom auth system to apache using [authnz_external](https://github.com/phokz/mod-auth-external/tree/master/mod_authnz_external) module.

I have written a server that authenticates a user via http request and Basic Auth.  So, it works as follows:
```
curl -u himanshug https://auth.expresshandbook.com
Password: ****
```
Now I want to hook this to apache so that any request to apache is first authenticated and then the request should be served.

Here is how I did that.

1. Install apache and mod_authnz_external
```
$ sudo apt-get install apache2 libapache2-mod-authnz-external
```
I had apache2.4 installed

2. Edit apache configuration to use the external authentication
   Edit `/etc/apache2/sites-enabled/000-default.conf`
```
DefineExternalAuth basic-auth pipe /usr/local/bin/basic-auth

<VirtualHost *:80>
Something...
</VirtualHost>

<Directory "/var/www/html">
    AuthType Basic
    AuthName basic-auth
    AuthBasicProvider external
    AuthExternal basic-auth
    Require valid-user
</Directory>
```

Let's understand meaning of definitions.

`DefineExternalAuth basic-auth pipe /usr/local/bin/basic-auth` - This defines an auth handler.

`basic-auth` is the name of handler.  You can choose any name.  

`pipe` is the mechanism how to pass the credentials to the external program.  pipe means that credentials are passed via stdin.  (Like echo $creds | <program>).

`/usr/local/bin/basic-auth` is a program that does a HTTP request to actual auth system and returns response to apache.  It can be written in any language (PHP, Python, Perl) and I have chosen Python.

3. Reload apache
```
$ sudo systemctl reload apache2
```

4. Write the program `/usr/local/bin/basic-auth`
```
#!/usr/bin/env python

import sys
import requests

creds = []
for l in sys.stdin:
    creds.append(l.strip()) # First line has the username and second the password

username = creds[0]
password = creds[1]
resp = requests.get("https://auth.expresshandbook.com/role/any", auth=(username, password))
if resp.status_code == 200:
    exit(0) # Successfull auth response to apache

exit(1) # Unauthenticated response to apache (non zero exit code)
```

Now just open "localhost" and it should ask for credentials and if successfull, it will open page.  Check /var/log/apache2/error.log if it does not work.

