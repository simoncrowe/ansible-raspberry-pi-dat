## ansible-raspberry-pi-dat
This repo automates setting up a Raspberry Pi as a DAT web server running 
Homebase.

It is partly inspired by https://github.com/agoramaHub/ansible-raspberry-server. 
I started a new repo largely because of the lack of a licence.

### Instructions
Clone this repository.
```bash
git clone https://github.com/simoncrowe/ansible-raspberry-pi-dat-homebase.git
cd ansible-raspberry-pi-dat-homebase
```

To use any of these playbooks you will need Ansible installed. One easy way, 
though it does require [pipenv](https://pipenv-es.readthedocs.io/es/stable/#install-pipenv-today), 
is to use the PipFile included. 
```bash
pipenv install
pipenv shell
```
After running the above you should be able to run Ansible from inside 
a newly created Python virtual environment.

#### 1. SSH Authentication Setup
##### 1.1. Preparation
You'll need to install the
[sshpass](https://www.tecmint.com/sshpass-non-interactive-ssh-login-shell-script-ssh-password/) 
package for this step to work. If you haven't used the pipenv method above,
you'll also need the 
[passlib](https://passlib.readthedocs.io/en/stable/install.html#installation-instructions) 
PIP package.

If you haven't already generated ssh keys for your machine (not the pi), 
you can do so with the [ssh-keygen](https://www.ssh.com/ssh/keygen/) shell 
command.

You'll need an 
[SSH-enabled](https://www.raspberrypi.org/documentation/remote-access/ssh/) Pi 
with a fresh Raspbian installation. Raspbian Lite makes more sense for a server.

Power up your Pi and ensure it's connected to your network. 
Ethernet is preferable; 
[WiFi](https://www.raspberrypi.org/documentation/configuration/wireless/README.md) 
is also an option. 

If you have only one Pi connected to your network and the following command 
is successful, you can proceed to step 1.2.

```bash
ping raspberrypi.local
```

Successful output should look something like this:

```bash
64 bytes from 192.168.1.3: icmp_seq=1 ttl=64 time=2.16 ms
```

If you do see something like the above, it may be worth replacing 
`raspberrypi.local` with pi's IP address (e.g. `192.168.1.3`) to the file called
`hosts` in the root directory of this repository. The hostname 
`raspberrypi.local` isn't that reliable and it could cause problems with Ansible 
later.

Unsuccessful output will look like this:

```bash
ping: unknown host raspberrypi.local
```

If you get this, your Pi might not reachable on your network or only by its IP 
address. If you have more than one Pi connected, you'll still need to find the 
IP address of the Pi you want to set up. 

<details>
<summary>One way to see if your Pi is on your network is using nmap</summary>

If you don't have nmap installed, you should be able to get it via your
system package manager.  e.g. `sudo apt install nmap`

This command will thoroughly scan your local network and may take several 
minutes.
```bash
sudo nmap -A 192.168.1.1/24
```
If your Pi is connected, its report should look something like this:
```
...
Nmap scan report for 192.168.1.3
Host is up (0.00091s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Raspbian 10 (protocol 2.0)
| ssh-hostkey: 
|   2048 ba:88:1f:54:0f:61:10:34:98:f4:5c:f2:35:79:cd:4f (RSA)
|_  256 68:92:a4:8e:da:b3:65:89:23:a3:3d:49:9c:a9:ab:9f (ECDSA)
MAC Address: DC:A6:32:67:9F:6E (Unknown)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.0
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
...
```
The line `22/tcp open  ssh     OpenSSH 7.9p1 Raspbian 10 (protocol 2.0)` 
will only appear is your Pi has SSH enabled. If you can't easily identify your 
Pi, double-check that SSH has been enabled on it.

If you see more than one Pi, you'll need to either temporally switch off your
Pi to work out which one it is, or find out its MAC address.
</details>

If your Pi does turn out not to be accessible at `raspberrypi.local`, you'll 
need to put its IP address in the file called `hosts` in the root directory 
of this repository.

##### 1.2. Automated setup 

The next step is to run the relevant playbook.
```bash
ansible-playbook auth-init.yaml -u pi --ask-pass
```
You'll prompted four things:
1. `SSH password:` the password of the default `pi` user on your Pi. 
if you haven't changed this yet, it'll be `raspberry`.
2. The path to your public SSH key. If you don't know, 
it's probably the default, so hitting ENTER is fine.
3. The new password for the default `pi` user of your Pi. It's a good idea to 
change this from the default, so let's automate it!

If the playbook runs to completion without errors, you will no longer 
be prompted for a password when opening an SSH session on your Pi. E.g.
```bash
ssh pi@raspberrypi.local
```

You will, however, not be able to SSH into your PI using another machine with 
a different public key to the one you've run the playbook on. If you want to 
authenticate from other machines, you can 
[set this up manually](https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md)

#### 2. Server Setup
If step 1 was successful, setting up DAT/Homebase should be simple.

First, install the third-party roles:
```bash
ansible-galaxy install -r requirements.yaml
```

Second, run the main playbook:
```bash
ansible-playbook site.yaml
```

If successful, the above command applies a number of Ansible roles listed in 
site.yaml to your Pi, including basic security configuration. 
A consequence of this hardened security is that you will now be unable to 
SSH into your Pi using a user's password.
Passwordless login will still work, and you can 
[manually](https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md) 
add the public keys of other machines to your Pi if needed.

#### 3. Server Modification

You'll need to SSH into your pi to change the default Homebase configuration:

```
ssh pi@<hostname or IP address of pi>
```
You may be able to connect to the hostname `raspberrypi.local`. Failing at that
you'll need to use the pi's IP address.

Now edit the Homebase configuration file, following the Homebase Docs:
```bash
vim .homebase.yml
```

You can run homebase in the foreground to see if there are any errors:
```bash
homebase
```

Once you're happy with your config you can run it daemonised using:
```bash
pm2 start homebase
```

Similarly, you can stop it using:
```bash
pm2 stop homebase
```
