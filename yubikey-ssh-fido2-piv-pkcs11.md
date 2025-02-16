# Hardware-Tokens in conjunction with SSH, PIV, GPG and other stuff

## Getting FIDO2 backed SSH to work
(aka how to solve `sign_and_send_pubkey: signing failed for ED25519-SK "ssh:" from agent: agent refused operation` when using a resident SSH-key on a Yubikey (usually created using `ssh-keygen -t ed25519-sk -O resident -O verify-required -C "comment`)

### Root of the problem
If all else looks good (see below), then as of Feb 2025 the problem is caused by the Gnome Keyring Daemon's ssh integration. For some reason it by default overrides openSSH's ssh-agent with its own functionality, which does not support hardware-tokens with a pin (or some such, didn't dig into the details there). Can be solved in different ways. All of this is assuming you already ran `ssh-keygen -K`, such that there exists a file (usually named `id_ed25519_sk_rk` in your ~/.ssh/ folder.
1. (affects only one function call - no changes to the system) : disable any agent integration for the current ssh session by adding the `-o "IdentityAgent=none"` flag and referencing to the pseudo private key (which only points to the HW-key) directly, e.g.:
   ```ssh -p port -i ~/.ssh/id_ed25519_sk_rk -o "IdentityAgent=none" user@host```  
   [source](https://www.reddit.com/r/yubikey/comments/wip57i/comment/ijfw8bg/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button)
2. run `gnome-session-properties` and disable gnome-keyring's ssh integration  (yet untested)  
   [source](https://wiki.gnome.org/Projects/GnomeKeyring/Ssh)
5. edit/create `~/.ssh/config` (for user-based config) or `/etc/ssh/ssh_config` (for machine wide config) and specify the `IdentityAgent=xxx` in there  (yet untested)  
   [source](https://man.openbsd.org/ssh_config.5)


### General checklist for this issue
1. File-permissions on ~/.ssh and the files therein
2. using openSSH 8.3 or above (I think)
3. 
