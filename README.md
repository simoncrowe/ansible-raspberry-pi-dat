## ansible-raspberry-pi-dat
This repo automates setting up a Raspberry Pi as a DAT web server running 
Homebase with the following basic features:
- Pin one DAT with a HTTPS mirror.
- Pin as many DATs as you need without HTTPS mirroring.

You should be able to modify the `homebase` role to enable more features.

<details>
<summary>Why only one HTTPS-mirrored DAT?</summary>
I was unable to get Homebase's own Letsencrypt SSL certificate provisioning 
feature working. Instead I've used a combination of NGINX and Certbot.

Homebase listens for HTTP requests on port 8080 and NGINX acts as a reverse proxy,
enabling HTTPS requests to be passed to Homebase. Because the NGINX server 
proxies localhost:8080, only one DAT can be mirrored to HTTPS.

If you know of a better solution, please let me know or open a PR.
</details>
This Ansible approach is inspired by https://github.com/agoramaHub/ansible-raspberry-server. 
I started a new repo largely because of the lack of a licence in agoramaHub's one.

### Instructions
Clone this repository.
```bash
git clone https://github.com/simoncrowe/ansible-raspberry-pi-dat-homebase.git
cd ansible-raspberry-pi-dat-homebase
```

To use any of these playbooks you will need Ansible installed. (You may wish to do this in a [virtual environment](https://docs.python.org/3/tutorial/venv.html).)

```bash
pip install ansible passlib
```

#### 1. SSH Authentication Setup
##### 1.1. Preparation
You'll need to install the
[sshpass](https://www.tecmint.com/sshpass-non-interactive-ssh-login-shell-script-ssh-password/) 
package for this step to work.

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

##### 2.1. Configuration
You'll need to create a file called `private.yaml` in the `vars` subdirectory
of this repo and put the following YAML in it, replacing values as appropriate:

```yaml
# The url used to update dynamic DNS record (in case your IP address changes)
dynamic_dns_update_url: https://freedns.afraid.org/dynamic/update.php?sPAMSPAMSPAMSPAMSPAMSPAM=

# The DAT that you want to mirror over HTTPS
landing_page_dat_url: dat://c6bbbb7c3f292ddca9df3c6ebcdb9c21a66a3f0d3dad065cbfb0a59bb0098aa3/
# The domain name that you want to mirror the above DAT on.
# This must have a DNS record pointing at your Pi's IP address.
landing_page_https_domain: example.com
# The email address to use when verifying SSL certificates with Letsencrypt
letsencrypt_email: info@example.com

# Add any other dat URLs you want to pin using Homebase in this list
hosted_dat_urls:
  - dat://bc9fc525239efd6e886a4b7d402ee800d1dd2812363f3be5161f0423fa46d3a3
  - dat://c57ef9a28674ff072d293ac744a172a2aa4c975ea8ffeba964fed23fbca2ce77
# If you don't want to pin any, just specify an empty list like so:
# hosted_dat_urls: []
```
##### 2.2. Setup
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

You'll need to SSH into your pi to start the Homebase service:

```
ssh pi@<hostname or IP address of pi>
```
You may be able to connect to the hostname `raspberrypi.local`. Failing at that
you'll need to use the pi's IP address.


You can run Homebase as a background process (deamon) using pm2:
```bash
pm2 start homebase
```

Similarly, you can stop it using:
```bash
pm2 stop homebase
```
If homebase isn't working, or running in the foreground to see if there are any errors:
```bash
homebase
```
