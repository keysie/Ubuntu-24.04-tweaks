# Resolving the issue of Ubuntu not resolving xxx.local domains

## TL;DR:
```
sudo rm -f /etc/resolv.conf
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
```
[source](https://askubuntu.com/a/1114887)
