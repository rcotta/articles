# Quickly creating a secured Squid proxy on AWS

## Problem

A user needs to access a given target system under one of these conditions:

* Target system uses odd network ports (i.e. 9002, 8088, etc), but corporate network rules restrict internet access to some common ports like 443 or 80
* Target system uses IP address whitelisting, but user's IP is dynamic
* Target system is blacklisted by the corporate proxy

## Proposed solution

The solution proposed is to create a low-cost proxy using Amazon Web Services (AWS) EC2, with access restricted by SSH keys and data encryption done also by the SSH mechanism.

Although the step by step provided is based on AWS and Amazon Linux, you'll probably be able to follow the same approach using other cloud providers and other Linux distributions.

## Step by step solution

### Create AWS machine

1. go to [AWS console](https://console.aws.amazon.com) and log in
1. create a new EC2 instance, by finding "EC2" option on the services menu, click the "launch instance" item and select the Amazon Linux AMI 2 flavor
1. follow the steps provided to create the new machine, choosing the options accordingly to your needs (default options will probably be enough; in my case, I reviewed the disk size to 15GB)
1. during the process remember to download the key pairs provided to access your EC2 instance
1. assign an Elastic IP to the new machine:
  1. on the left menu find "Network & Security / Elastic IPs"
  1. click on the "Allocate new address" button
  1. select the option "Amazon pool" and click "Allocate"
  1. find your running instance on EC2 console, right click and associate the new IP address created
  
### Install Squid proxy

1. login to your new machine using the ssh key downloaded (on Windows I suggest you to use [Putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html))
1. install Squid by escalating your access to su (`sudo su`) and then installing through yum (`yum install squid`; provide the confirmations when prompted)
1. default configuration (`/etc/squid/squid.conf`) will only allow access from localhost and local network addresses, but consider reviewing settings file (*IMPORTANT:* we do want to keep access restricted to localhost!)
1. if some setting is changed, then remember to restart Squid (`/sbin/service squid restart`)

### Change SSH daemon to listen to 443 port

1. edit sshd config file (`/etc/ssh/sshd_config`)
1. add the line `Port 443`
1. restart sshd service (`/sbin/service sshd restart`)

### Create the users and configure access

We'll need to provide SSH access to the users. If the proxy server is for your use only, you may want to use the keys provided on the EC2 instance creation, if not you may want to create individual users as described:

1. as `su` on the EC2, add a new user (`adduser new.user --disabled-password`; replace `new.user` by the login to your new user)
1. change the current user to the new user and create the key pair (alert: copy & paste of the commands I did; if you find some unnecessary command let me know):
  ```
  su new.user
  cd ~
  mkdir .ssh
  chmod 700 .ssh
  touch .ssh/authorized_keys
  chmod 600 .ssh/authorized_keys
  ssh-keygen
  cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
  ```
1. you may want to `exit` and add more users.

If you prefer to create password-protected accounts, then you can follow the normal `useradd` and `passwd` flow.

### Access the machine and the reverse proxy

Here's the more curious part: we'll access the machine through SSH running on port 443 (i.e. to bypass corporate proxy restrictions), and we'll have access to Squid through port forward.

1. if you're using Linux, use `ssh -L 3128:127.0.0.1:3128 new.user@elastic-ip-address -p 443`, paying attention to replace `new.user` and `elastic-ip-address` to the correct values
1. if you're using Putty on Windows, user `new.user@elastic-ip-address` on host name, on "SSH/Auth" select the private key file* on "SSH/Tunnels" use source `port = 443`, `destination = 127.0.0.1:3128` 

(*) on Putty, to generate the private key file, copy the contents of `~/.ssh/id_rsa` file, save on Windows, use Puttygen application to open it and choose "save private key" option; this will make the conversion from the original format to the .ppk format used by Putty

Finally, it's time to find the proxy settings on your browser and set it to 127.0.0.1 port 3128.

## Final considerations

To validate if everything is in place, you can access [https://www.whatsmyip.org/](https://www.whatsmyip.org/) and check if the IP address reported and the Elastic IP created are the same.

The only inconvenient is that you'll need to keep the ssh connection active while browsing the internet.

I don't intend to explain this approach to any illegal purposes. Also, consider that provisioning AWS resources involve costs.
