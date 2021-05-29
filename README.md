This repo automates setting up a Raspberry Pi as a hyper/https server. 
It uses [Homebase](https://github.com/beakerbrowser/homebase) to pin hyperdrves 
and NGINX to securely handle requests to Homebase's HTTP service.
with the following basic features:
- Pin one hyperdrive with a HTTPS mirror.
- Pin as many hyperdrives as you need without HTTPS mirroring.

You should be able to modify the `homebase` role to enable more features.

<details>
<summary>Why only one HTTPS-mirrored hyperdrive?</summary>
Version 3 of Homebase has dropped the Letsencrypt feature.
In its place I've used a combination of NGINX and Certbot.

Homebase listens for HTTP requests on port `8080` and NGINX acts as a reverse proxy,
enabling HTTPS requests to be passed to Homebase.
Homebase uses the host `localhost` and NGINX routes requests to `localhost:8080`;
as there's only one localhost, only one hyperdrive can be mirrored to HTTPS.

I have a vague idea of what virtual hosts are. If you need this feature,
please let me know or open a PR.
</details>

### Instructions
Clone this repository.
```bash
git clone https://github.com/simoncrowe/ansible-raspberry-pi-hyperdrive-homebase-nginx.git
cd ansible-raspberry-pi-hyperdrive-homebase-nginx
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

If you haven't already generated ssh keys for your machine (not the Pi),
you can do so with the [ssh-keygen](https://www.ssh.com/ssh/keygen/) shell
command.

You'll need a pi with a fresh [Raspberry Pi OS](https://www.raspberrypi.org/downloads/raspberry-pi-os/)
(a.k.a. Raspbian) installation.
The _Lite_ version of the OS makes more sense for a server as it lacks a GUI.

Enable SSH on the pi. You should follow section 3 of this
[page](https://www.raspberrypi.org/documentation/remote-access/ssh/)
for headless Raspberry Pi if you opted for _Raspberry Pi OS Lite_.

Power up your pi and ensure it's connected to your network.
Ethernet is preferable;
[WiFi](https://www.raspberrypi.org/documentation/configuration/wireless/README.md)
is also an option.

If you have only one pi connected to your network and the following command
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
`raspberrypi.local` isn't very reliable and it could cause problems with Ansible
later.

Unsuccessful output will look like this:

```bash
ping: unknown host raspberrypi.local
```

If you get this, your pi might not reachable on your network or only by its IP
address. If you have more than one pi connected, you'll still need to find the
IP address of the pi you want to set up.

<details>
<summary>One way to see if your pi is on your network is using nmap</summary>

If you don't have nmap installed, you should be able to get it via your
system package manager.  e.g. `sudo apt install nmap`

This command will thoroughly scan your local network and may take several
minutes.
```bash
sudo nmap -A 192.168.1.1/24
```
If your pi is connected, its report should look something like this:
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
will only appear is your pi has SSH enabled. If you can't easily identify your 
pi, double-check that SSH has been enabled on it.

If you see more than one pi, you'll need to either temporally switch off your
pi to work out which one it is, or find out its MAC address.
</details>

If your pi does turn out not to be accessible at `raspberrypi.local`, you'll 
need to put its IP address in the file called `hosts` in the root directory 
of this repository.

##### 1.2. Automated setup 

The next step is to run the Ansible playbook for setting up SSH 
authentication on your pi.
```bash
ansible-playbook auth-init.yaml -u pi --ask-pass
```
You'll prompted three things:
1. `SSH password:` the password of the default `pi` user on your pi. 
if you haven't changed this yet, it'll be `raspberry`.
2. The path to your public SSH key. If you don't know, 
it's probably the default, so hitting ENTER is fine.
3. The new password for the default `pi` user of your pi. It's a good idea to 
change this from the default, so let's automate it!

If the playbook runs to completion without errors, you will no longer
be prompted for a password when opening an SSH session on your pi. E.g.
```bash
ssh pi@raspberrypi.local
```

You will, however, not be able to SSH into your pi using another machine with
a different public key to the one you've run the playbook on. If you want to
authenticate from other machines, you can
[set this up manually](https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md)

#### 2. Server Setup
If step 1 was successful, setting up Homebase should be simple.

##### 2.1. Configuration
You'll need to create a file called `private.yaml` in the `vars` subdirectory
of this repo and put the following YAML in it, replacing values as appropriate:

```yaml
# The url used to update dynamic DNS record (in case your IP address changes)
dynamic_dns_update_url: https://freedns.afraid.org/dynamic/update.php?sPAMSPAMSPAMSPAMSPAMSPAM=

# The hyperdrive that you want to mirror over HTTPS
landing_page_hyper_url: hyper://c6bbbb7c3f292ddca9df3c6ebcdb9c21a66a3f0d3dad065cbfb0a59bb0098aa3/
# The domain name that you want to mirror the above hyperdrive on.
# This must have a DNS record pointing at your pi's IP address.
landing_page_https_domain: example.com
# The email address to use when verifying SSL certificates with Letsencrypt
letsencrypt_email: info@example.com

# Add any other hyperdrives you want to pin using Homebase in this list
hosted_hyper_urls:
  - hyper://bc9fc525239efd6e886a4b7d402ee800d1dd2812363f3be5161f0423fa46d3a3
  - hyper://c57ef9a28674ff072d293ac744a172a2aa4c975ea8ffeba964fed23fbca2ce77
# If you don't want to pin any more hyperdrives, just specify an empty list like so:
# hosted_hyper_urls: []
```
##### 2.2. Setup
First, install the third-party roles:
```bash
ansible-galaxy install -r requirements.yaml
```

Second, run the main playbook:
```bash
ansible-playbook setup-hyper-server.yaml
```

If successful, the above command applies a number of Ansible roles listed in
site.yaml to your pi, including basic security configuration.
A consequence of this hardened security is that you will now be unable to
SSH into your pi using a user's password
Passwordless login will still work, and you can
[manually](https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md)
add the public keys of other machines to your pi if needed.

#### 3. Runnning Homebase

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
If your Hyperdrive doesn't get a new peer connected when you start homebase and/or
you can't access the drive over HTTPS, try running in the foreground to see if there are any errors:
```bash
homebase
```

#### 4. Using PiJuice HAT (optional)

This sections deal with managing a connected PiJuice HAT.

##### 4.1 Install PiJuice software
Again, you'll need to SSH into your pi:
```
ssh pi@<hostname or IP address of pi>
```

Once connected to the pi, you can install the `pijuice-base` package.
```bash
sudo apt-get install -y pijuice-base
```

##### 4.2 Use the CLI
You should be able to configre a connected HAT using the
PiJuice command line interface. Start the CLI like so:
```bash
pijuice_cli
```
Instructions can be found [here](https://github.com/PiSupply/PiJuice/tree/master/Software#pijuice-cli)
