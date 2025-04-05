# Getting FIDO2 backed SSH to work
(aka how to solve `sign_and_send_pubkey: signing failed for ED25519-SK "ssh:" from agent: agent refused operation` when using a resident SSH-key on a Yubikey (usually created using `ssh-keygen -t ed25519-sk -O resident -O verify-required -C "comment`)

### Root(s) of the problem
1. Permissions of ~/.ssh/ and the files therein.
2. Version of OpenSSH must be 8.3 or above
3. ```ssh-askpass-gnome``` must be installed, otherwise ssh-agent is unable to query the device PIN. Annoyingly it does not really announce this propperly. All you get is ssh-agent debug output like ```incorrect passphrase supplied to decrypt private key```. This one took me forever to figure out.
4. If all else looks good (see below), then as of Feb 2025 the problem is caused by the Gnome Keyring Daemon's ssh integration. For some reason it by default overrides openSSH's ssh-agent with its own functionality, which does not support hardware-tokens with a pin (or some such, didn't dig into the details there).

There seems to exist an [issue with the gnome-keyring project](https://gitlab.gnome.org/GNOME/gnome-keyring/-/issues/101) over on GitLab, but noone has touched that in years.

### Solutions
Can be solved in different ways. All of this is assuming you already ran `ssh-keygen -K`, such that there exists a file (usually named `id_ed25519_sk_rk` in your ~/.ssh/ folder.

- #### Quick & Dirty:   
   For debugging or some one-time session, simply disable any agent integration for the current ssh session by adding the `-o "IdentityAgent=none"` flag and reference the pseudo private key (which only points to the HW-key) directly, e.g.:  
   ```ssh -p port -i ~/.ssh/id_ed25519_sk_rk -o "IdentityAgent=none" user@host```  
   --> [source](https://www.reddit.com/r/yubikey/comments/wip57i/comment/ijfw8bg/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button)
  
- #### Current shell session:
   A more convenient solution, which doesn't affect the system in a permanent way, is to set/overwrite the SSH_AUTH_SOCK environment variable by using the output of `ssh-agent -s`. This is most easily done by running:  
   ```eval $(ssh-agent -s)```   
   This will tell ssh to use the openSSH ssh-agent for the rest of the current shell session. You'll maybe need to run `ssh-add -K` again to add the identity to the correct agent. Simply use ssh-add and ssh-keygen from this point forward (during this shell session), and it will always be the correct one. To debug this, one can simply `echo $SSH_AUTH_SOCK`. If the output look smth like `/run/user/xxxx/keyring/ssh`, then the GNOME keyring is still being used. If it looks like `/tmp/ssh-xxxxxxxx/agent.xxx`, then the openSSH agent is actively being used.

  Read [this write-up](https://unix.stackexchange.com/a/338200) to get some understanding of which service sets and overwrites SSH_AUTH_SOCK in the process of system start-up and user log-in

- #### Permanent solution:
  After some trial and error I found, that disabling gnome-keyring-ssh (see below) does not suffice to make the propper ssh-agent start and set SSH_AUTH_SOCK for interactive terminal sessions. I also fiddled around with adding scripts to /etc/profile.d/ to get these changes to be machine-wide, but it seems that gnome still overwrites SSH_AUTH_SOCK after the /etc/profile.d/-scripts have been executed. The only way I found so far to get this to be permanent is to add the function below to the user's ~/.bashrc. Simply running `eval "$(ssh-agent -s)"` in .bashrc is not a good solution, since every execution of that command is starting a new instance of ssh-agent. The function solves this problem very elegantly by iterating over all open ssh-sockets and checking which ones work. If none exist, one instance of ssh-agent is started. The solution isn't mine, so full credit goes to the [source](https://gist.github.com/tomquisel/303fc37d0e854d59b672?permalink_comment_id=4061996#gistcomment-4061996).

  One small change I have made is to remove the SSH_AUTH_SOCK variable from the list of sockets to be checked, in order to ensure the gnome keyring is ignored.
  ```
  function setup_ssh_agent {
	if
		echo "List of running SSH agents:"
		pgrep --list-full --uid $USER --exact ssh-agent
	then
		local oldnullglob="$(shopt -p nullglob)"
		shopt -s nullglob
		local -a try_socks=(/tmp/ssh-*/agent.*)
		$oldnullglob

		local txt
		for sock in "${try_socks[@]}"
		do
			echo -n "Trying $sock ... "
			export SSH_AUTH_SOCK=$sock
			if
				txt=$(ssh-add -l) ||
				[[ "$txt" == "The agent has no identities." ]]
			then
				echo "OK"
				return
			else
				echo
			fi
		done

		echo "A working SSH Agent socket could not be found"
		return 1
	else
		echo -n "None found. Starting one... "
		. <(ssh-agent)
	fi
  }

  setup_ssh_agent
  clear
  ```
  This change makes it such that the same ssh-agent is used for all user terminals. This also means that in between reboots ssh-agent will remember added credentials. Reboot will cause ssh-agent to be re-initiated and thus it will forget any cached credentials. This can be solved with a small udev-rule (see below).

- #### BONUS: Add change to default user
  As sysadmin you'll probably not want to do this for every user manually. To make life easier, add the above changes to `/etc/skel/.bashrc`, such that new users automatically get it as part of their .bashrc.

- #### BONUS2: Make ssh-keys stay in ssh-agent between reboots
  Instead of just adding the identities of the resident keys to ssh-agent temporarily by using `ssh-add -K`, one can do the following (Yubikey needs to be plugged in):
  ```
  cd ~/.ssh/
  ssh-keygen -K
  ```
  This creates a public and a pseudo 'private' key file to the .ssh folder. No need to set a passphrase - the pseudo-private-keys are just pointers to the key material on the Yubikey (at least if PIN entry and touch is set to required). These identities can now be added to ssh-agent using `ssh-add ~/.ssh/<identity>`. This process can be further automated by adding that line to `.bashrc` in between `setup_ssh_agent` and `clear` like this:
  ```
  ...
  setup_ssh_agent
  ssh-add ~/.ssh/id_ed25519_sk_rk_XXXX
  ssh-add ~/.ssh/id_ed25519_sk_rk_YYYY
  clear
  ```

#### NOTE:
running `gnome-session-properties` and disabling gnome-keyring's ssh integration DOES turn that service off and keep it from interfering, however it DOES NOT fix the issue all by itself. If done this way, one must still find some way to tell ssh which ssh-agent to use (or reference the pseudo private key every single time). 
[source](https://wiki.gnome.org/Projects/GnomeKeyring/Ssh)


### General checklist for this issue
1. File-permissions on ~/.ssh and the files therein
2. using openSSH 8.3 or above (I think)
3. 
