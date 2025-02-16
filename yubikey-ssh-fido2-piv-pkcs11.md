# Hardware-Tokens in conjunction with SSH, PIV, GPG and other stuff

## Getting FIDO2 backed SSH to work
(aka how to solve `sign_and_send_pubkey: signing failed for ED25519-SK "ssh:" from agent: agent refused operation` when using a resident SSH-key on a Yubikey (usually created using `ssh-keygen -t ed25519-sk -O resident -O verify-required -C "comment`)

### Root of the problem
If all else looks good (see below), then as of Feb 2025 the problem is caused by the Gnome Keyring Daemon's ssh integration. For some reason it by default overrides openSSH's ssh-agent with its own functionality, which does not support hardware-tokens with a pin (or some such, didn't dig into the details there). 

### Solutions
Can be solved in different ways. All of this is assuming you already ran `ssh-keygen -K`, such that there exists a file (usually named `id_ed25519_sk_rk` in your ~/.ssh/ folder.
1. (affects only one function call - no changes to the system):   
   Disable any agent integration for the current ssh session by adding the `-o "IdentityAgent=none"` flag and referencing to the pseudo private key (which only points to the HW-key) directly, e.g.:  
   ```ssh -p port -i ~/.ssh/id_ed25519_sk_rk -o "IdentityAgent=none" user@host```  
   --> [source](https://www.reddit.com/r/yubikey/comments/wip57i/comment/ijfw8bg/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button)
3. (affects only the current shell session):  
   Set/overwrite the SSH_AUTH_SOCK environment variable either manually, or by using the output of `ssh-agent -s`. This is most easily done by running:  
   ```eval $(ssh-agent -s)```   
   This will tell ssh to use the openSSH ssh-agent for the rest of the current shell session. You'll maybe need to run `ssh-add -K` again to add the identity to the correct agent. Simply use ssh-add and ssh-keygen from this point forward (during this shell session), and it will always be the correct one. To debug this, one can simply `echo $SSH_AUTH_SOCK`. If the output look smth like `/run/user/xxxx/keyring/ssh`, then the GNOME keyring is still being used. If it looks like `/tmp/ssh-xxxxxxxx/agent.xxx`, then the openSSH agent is actively being used.
5. edit/create `~/.ssh/config` (for user-based config) or `/etc/ssh/ssh_config` (for machine wide config) and specify the `IdentityAgent=xxx` in there  (yet untested)  
   [source](https://man.openbsd.org/ssh_config.5)

#### NOTE:
running `gnome-session-properties` and disabling gnome-keyring's ssh integration DOES turn that service off and keep it from interfering, however it DOES NOT fix the issue all by itself. If done this way, one must still find some way to tell ssh which ssh-agent to use (or reference the pseudo private key every single time). 
[source](https://wiki.gnome.org/Projects/GnomeKeyring/Ssh)


### General checklist for this issue
1. File-permissions on ~/.ssh and the files therein
2. using openSSH 8.3 or above (I think)
3. 
