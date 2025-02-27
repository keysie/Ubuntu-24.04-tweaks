# Ubuntu-24.04-tweaks

## Goal
This repository is just a growing set of guides and loose documents regarding often used hacks, workarounds and just more complex processes often needed to get stuff to work under Ubuntu. Each file is dedicated to one topic. 

## Topics:
- #### [SSH with resident FIDO2 hardware keys on Ubuntu](https://github.com/keysie/Ubuntu-24.04-tweaks/blob/main/yubikey-ssh-fido2.md)
  TL;DR: Fix `sign_and_send_pubkey: signing failed for ED25519-SK` when using resident FIDO2 ssh keys on a device like a Yubikey by disabling the gnome keyring ssh integration. Make the fix permanent using a function added to `.bashrc`.
- #### [DNS not resolving *.local domains](https://github.com/keysie/Ubuntu-24.04-tweaks/blob/main/resolving-local-domain.md)
  TL;DR: `sudo rm -f /etc/resolv.conf && sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf`
