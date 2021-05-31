# Gain full access to your remote MAD-ATV Boxes

The information in this repository will enable you to gain full ADB (and thus tools based on it, especially `scrcpy`) access to MAD-ATV boxes in remote locations utilizing only a MAD job. We will use `dropbear`, a SSH replacement tailored for weaker devices, on the ATV box to connect to your server via SSH and create a tunnel from a local port on your server to the ADB service on the ATV box. Thus, you will be able to connect to the ADB service on your server using `adb connect localhost:<port>`.

This has only been tested on ATV boxes running the 64bit MAD ROM, because I don't own any other scanning devices. Judging by the naming of the archive I got the `dropbearmulti` binary from, it **could possibly** also run on 32bit devices.

## Security warning

Please do not follow these instructions if you're unsure about possible security implications, especially since we'll be transferring an unprotected private key to remote ATV boxes enabling anyone with access to these boxes to obtain the private key and connect to the SSH service on your server/machine. I'm an amateur in all these things and while I did my best to Google the appropriate security measures and the user *should* not be able to do any harm to your server as far as I am aware, I cannot guarantee the security of these instructions. **Proceed at your own risk!**

## Requirements

* remotely accessible SSH service on your server and basic understanding of how to connect and interact with it
* this guide is assuming a recent Ubuntu (server) installation
* ability to (securely) make files available for download from your server
* ADB installed on your server

# Guide

## Install required software

In order to create a dropbear compatible keypair, we need the `dropbearkey` program. In Ubuntu, this is available through the `dropbear` package (`apt install dropbear`).

## Create local user without shell

Create a new local user on your server with `sudo useradd -m remotemaduser --shell /bin/false`. Defining the shell as `/bin/false` is one part of ensuring this user won't be able to do anything other than creating our tunnel when connecting through SSH.

* verify the user's home directory exists by checking `ls -la /home`
* verify the passwd entry has the correct info: `grep mad /etc/passwd` should return this (user ID 1002 can of course differ): `remotemaduser:x:1002:1002::/home/remotemaduser:/bin/false`

## Create dropbear keypair in the new user's .ssh directory

* you will need to execute these steps as root user: `sudo su`
* create a .ssh folder and move into it: `cd /home/remotemaduser; mkdir .ssh; cd .ssh`
* run: `dropbearkey -t rsa -s 1024 -f keyfilename`. This will create a private key in the file `keyfilename` and print the public key directly to your console after the text `Public key portion is:`. The public key starts with `ssh-rsa` and ends with your local identification info like `username@hostname`. Copy everything **including** `ssh-rsa` and the identifier `username@hostname` and store it in a file like `keyfilename.pub` in the current `/home/remotemaduser/.ssh/` directory.
* move the `keyfilename` private key (NOT the `.pub` public key) somewhere to make it downloadable through the internet, preferrable with basic auth. This is out of the scope of this guide. Mind proper file ownership and permissions.
* setup ownership and permissions for the folder structure and `.pub` file: `chmod 600 keyfilename.pub; chown -R remotemaduser:remotemaduser /home/remotemaduser`
* exit the root shell: `exit`

## Change sshd config

Add the following configuration to `/etc/ssh/sshd_config`, replace the port `12345` with whichever port you want the ADB service to be available on:
```
Match User remotemaduser
  PermitOpen 127.0.0.1:12345
  X11Forwarding no
  AllowAgentForwarding no
  PasswordAuthentication no
  ForceCommand /bin/false
  AuthorizedKeysFile /home/remotemaduser/.ssh/keyfilename.pub
  RSAAuthentication yes
  PubkeyAuthentication yes
  ```
  
Restart your SSH server.

## Verify SSH setup

At this point, you should be able to connect locally with the files we created. Attempt the following:
`dbclient remotemaduser@localhost -i /whereever/you/moved/the/keyfilename -R 12345:localhost:12346`

This should exit within a second or two without any output. **If you get any form of shell access through the `remotemaduser`, you did something wrong during the tutorial and have a security flaw at your hands.** In this case please go through the previous steps again and think about what you missed! If you're not sure what's happening at this point and how to proceed securely, revert the changes to your `/etc/ssh/sshd_config`, remove the `remotemaduser` user and its `/home/remotemaduser` directory **completely** and start over or leave it be to get professional support!

Now if the previous command exited automatically without any output, try: `dbclient remotemaduser@localhost -i /whereever/you/moved/the/keyfilename -R 12345:localhost:12346 -N -T` This should just silently block your shell, **not** exiting. 

In another shell, try `sudo netstat -tulpn | grep 1234` and confirm you have at least one entry like  ```tcp        0      0 127.0.0.1:12346         0.0.0.0:*               LISTEN      615647/sshd: remotemaduser```

If you found this entry in `netstat`, you can exit the silent `dbclient` in the first shell by pressing `CTRL+C`.

## Install and run the job

We should be all set now server-side! Now download the `madatv_full_remote_access.json` file form this repo, move it to your `MAD/personal_commands` folder and edit the parts marked with `<>` to match your setup, removing these `<>` in the process.

Go into MADmin, click Jobs -> Reload jobs, and run the `ATV - Start remote ADB` on one of your boxes! The job should end with status `success`, returning output similar to this (this is taken from a second run, the initial run will produce some other, but similar output):
```
[ln: cannot create symbolic link from '/data/local/tmp/dropbearmulti' to '/data/local/tmp/ssh': File exists, /data/local/tmp/ssh: Warning: failed to identify current user. Trying anyway., /data/local/tmp/ssh: Warning: failed creating //.ssh: Read-only file system, /data/local/tmp/ssh: , Host '1.2.3.4' key accepted unconditionally., (ssh-ed25519 fingerprint sha1!! a:b:c:d), ]
```

Now on your server, you should have full ADB access to this device, connecting with `adb connect localhost:<port>`! Congratulations!

Instead of connecting directly on your server, you could now proceed to further ssh-forward this port on your server to your local machine to adb connect and run scrcpy locally. Please Google a tutorial for this if you're unsure how to accomplish this on the OS of your choice.

After you're done, terminate the connection with the `ATV - Stop remote ADB` job. Only one device can be connected like this at a time.

## Bonus: Connecting to a third box

* You need to connect to a remote box, but it won't connect to MAD, so you can't run the `Start remote ADB` job on it?
* You have a second, fully working box right next to it?

Then you could do this:

* obtain the problematic boxes' local IP: ask your box host to check their router, or even better, have my [mp-activityFile](https://github.com/crhbetz/mp-activityFile) plugin running with its functionality to store local and public IP to a `.ips` file :-)
* edit the `madatv_full_remote_access.json` file (or create a copy), replacing `localhost:5555` with `localboxip:5555`
* reload jobs and run the new job on a working box at the same location

Now, your working box will create a tunnel from the problematic boxes' ADB service to your server, enabling you to connect to ADB on the problematic box!

# Disclaimers

## Dropbear binary

The `dropbearmulti` binary that's uploaded into this repository and used for the provided MAD job was obtained from `https://bitfab.org/dropbear-static-builds/`, direct link: `https://bitfab.org/dropbear-static-builds/dropbear-v2020.81-arm-none-linux-gnueabi-static.tgz`.
