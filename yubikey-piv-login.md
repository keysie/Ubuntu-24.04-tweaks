- assuming libykcs11.so is installed (check the other tweaks for details)
- install ```(opensc-pkcs11) pcscd sssd libpam-sss```
- create ```/etc/pkcs11/modules/libykcs11.so``` with content:
  ```
  module: /usr/local/lib/libykcs11.so
  critical: no
  priority: 100
  ```
- run ```p11-kit list-modules``` and check if ```module: libykcs11.so``` is the first response.
- [source](https://documentation.ubuntu.com/server/how-to/security/smart-card-authentication/)
