# How to get VS-Code to trust the self-signed cert of your firewall

Spoiler: It's not by adding the CA to the system using update-ca-certificates, and it's also not by installing chromium and adding it there. Why? Because Ubuntu will install the chromium snap - which of course is sandboxed and will not let vs-code use its certificate store.

Solution: (taken from https://chromium.googlesource.com/chromium/src/+show/refs/heads/main/docs/linux/cert_management.md)

## Details
### Get the tools
*   Debian/Ubuntu: `sudo apt install libnss3-tools`
*   Fedora: `sudo dnf install nss-tools`
*   Gentoo: `su -c  "echo 'dev-libs/nss utils' >> /etc/portage/package.use &&
    emerge dev-libs/nss"` (You need to launch all commands below with the `nss`
    prefix, e.g., `nsscertutil`.)
*   Opensuse: `sudo zypper install mozilla-nss-tools`
### List all certificates
    certutil -d sql:$HOME/.pki/nssdb -L
### List details of a certificate
    certutil -d sql:$HOME/.pki/nssdb -L -n <certificate nickname>
### Add a certificate
```shell
certutil -d sql:$HOME/.pki/nssdb -A -t <TRUSTARGS> -n <certificate nickname> \
-i <certificate filename>
```
The TRUSTARGS are three strings of zero or more alphabetic characters, separated
by commas. They define how the certificate should be trusted for SSL, email, and
object signing, and are explained in the
[certutil docs](https://firefox-source-docs.mozilla.org/security/nss/legacy/tools/nss_tools_certutil/index.html)
or
[Meena's blog post on trust flags](https://web.archive.org/web/20131212024426/https://blogs.oracle.com/meena/entry/notes_about_trust_flags).
For example, to trust a root CA certificate for issuing SSL server certificates,
use
```shell
certutil -d sql:$HOME/.pki/nssdb -A -t "C,," -n <certificate nickname> \
-i <certificate filename>
```
To import an intermediate CA certificate, use
```shell
certutil -d sql:$HOME/.pki/nssdb -A -t ",," -n <certificate nickname> \
-i <certificate filename>
```
Note: to trust a self-signed server certificate, we should use
```
certutil -d sql:$HOME/.pki/nssdb -A -t "P,," -n <certificate nickname> \
-i <certificate filename>
```
#### Add a personal certificate and private key for SSL client authentication
Use the command:
    pk12util -d sql:$HOME/.pki/nssdb -i PKCS12_file.p12
to import a personal certificate and private key stored in a PKCS #12 file. The
TRUSTARGS of the personal certificate will be set to "u,u,u".
### Delete a certificate
    certutil -d sql:$HOME/.pki/nssdb -D -n <certificate nickname>
