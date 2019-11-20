## ansible-raspberry-pi-dat
This repo automates setting up a Raspberry Pi as a DAT web server running 
Homebase.

It is partly inspired by https://github.com/agoramaHub/ansible-raspberry-server. 
I started a new repo largely because of the lack of a licence.

###Instructions
Clone this repository
```bash
git clone https://github.com/simoncrowe/ansible-raspberry-pi-dat-homebase.git
cd ansible-raspberry-pi-dat-homebase
```

To use any of these playbooks you will need Ansible installed. One easy way, 
though it does require [pipenv](https://pipenv-es.readthedocs.io/es/stable/#install-pipenv-today), is to use the PipFile included. 
```bash
pipenv install
pipenv shell
```
After running the above you should be able to run Ansible from inside 
a newly created Python virtual environment.

#### 1. SSH Authentication setup
You will probably need to install `sshpass` for this step to work.

If you haven't already generated ssh keys for your machine (not the pi), 
you can do so with the 'ssh-keygen` shell command.

You'll need a 
[SSH-enabled](https://www.raspberrypi.org/documentation/remote-access/ssh/) Pi 
with a fresh Raspbian installation. I'd recommend Raspbian Lite for a server.

Power up your Pi and ensure it's connected to your network. 
Ethernet is preferable; 
[WiFi](https://www.raspberrypi.org/documentation/configuration/wireless/README.md) 
is also an option. The following command should be successful.
```bash
ping raspberrypi.local
```

If it isn't your Pi may not be reachable on your network or only by its IP 
address. 

<details>
<summary>One way to see if your Pi is on your network is nmap</summary>

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
</details>

If your Pi does turn out not to be accessible at raspberrypi.local, you'll 
need to put its IP address in the file called `hosts` in the root directory 
of this repository.

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
4. Whether you want to disable password-based SSH authentication to you Pi
from now on. I'd recommend going with the default of `yes`. If you're exposing
a machine to the web, password authentication isn't that secure.

Hopefully the playbook runs to completion.