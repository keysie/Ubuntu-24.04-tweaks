# Get automatic DNS update working in windows AD

1. Update hostname of machine from somehostname to somehostname.your.domain (FQDN) using ```sudo hostnamectl set-hostname xxx.your.domain```
2. Also update the entry for 127.0.0.1 in ```/etc/hosts``` from ```127.0.0.1  somehostname``` to ```127.0.0.1  somehostname  somehostname.your.domain```
3. ```sudo systemctl restart sssd```
4. After a few seconds the dns-update should successfully terminate

[source](https://learn.microsoft.com/de-ch/archive/blogs/jeffbutte/265)
