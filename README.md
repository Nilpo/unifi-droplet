Setting up Ubiquiti Unifi on DigitalOcean
=========================================

## Creating a droplet on DigitalOcean

After logging in to the DigitalOcean [dashboard](https://cloud.digitalocean.com/droplets), click the **Create** button and choose **Droplets** from the expanding menu.

![screenshot](screenshots/unifi01.png)

 - Under **Choose an image**, select **One-click apps** and then choose **MongoDB 3.4.10 on 16.04**.

![screenshot](screenshots/unifi02.png)

 - Under **Choose a size**, select a size that fits your budget. (The smallest size should be just fine to get started. It's easy to scale your application up as you grow.)

![screenshot](screenshots/unifi03.png)

 - Under **Choose a datacenter region**, select a region that is close to you and your clients.
 - Under **Select additional options**, select both `Backups` and `Monitoring`. These free addons are important for any production system.
 - Under **Add your SSH keys**, select the SSH key for the machine you are currently working on or create one. If you choose not to use SSH authentication, you can still log in to your droplet using a username and password but SSH is the preferred method because of its increased security. More information about SSH logins can be found [here](https://www.digitalocean.com/community/tutorials/how-to-connect-to-your-droplet-with-ssh).

![screenshot](screenshots/unifi04.png)

 - Under **Finalize and create**, assign your droplet and appropriate name and click the `Create` button.

![screenshot](screenshots/unifi05.png)

Once your droplet is created, its details can be viewed in your droplets list. Make a note of the IP address that has been assigned to your droplet. You'll need this later along with the temporary password that you will receive in your welcome email.

![screenshot](screenshots/unifi06.png)

## Connect to your droplet

You will use SSH to connect to your new DigitalOcean droplet. For Linux and Mac systems, this is easiest done using OpenSSH at the command line. Windows users are recommended to use the free [PuTTY](http://www.putty.org/) tool.

Fill in your details on the PuTTY Configuration dialog. Pay special attention the the IP address. Make sure that it matches the IP address of your new droplet.

![screenshot](screenshots/unifi07.png)

Log in using the username `root` and the password that was supplied to you. When logging in for the first time, you will be required to set a permanent password for your root account. Once you have done this, you will be logged in to your droplet and you will have a root prompt.

![screenshot](screenshots/unifi09.png)

PuTTY will also warn you that it does not recognize the certificate for the server your are connecting to. Since this is your first time connecting, select `Yes` to accept the server's certificate and open the connection.

![screenshot](screenshots/unifi08.png)

To connect from the command line instead, use the following command:

	ssh root@<IP_ADDRESS>

![screenshot](screenshots/unifi10.png)

## Setting up your Ubuntu environment

### Add a regular account and disable login on the root account for security

 - Add a regular user account.

```
# adduser ubnt
Adding user `ubnt' ...
Adding new group `ubnt' (1000) ...
Adding new user `ubnt' (1000) with group `ubnt' ...
Creating home directory `/home/ubnt' ...
Copying files from `/etc/skel' ...
Enter new UNIX password: <YOUR_SECURE_PASSWORD>
Retype new UNIX password: <YOUR_SECURE_PASSWORD>
passwd: password updated successfully
Changing the user information for ubnt
Enter the new value, or press ENTER for the default
        Full Name []: Ubiquiti
        Room Number []: 
        Work Phone []: 
        Home Phone []: 
        Other []: 
Is the information correct? [Y/n] Y
```

 - Add the new user account to the sudoers group so that you can perform privileged commands.

```
# usermod -aG sudo ubnt
```

 - Log out and log back in using the newly created user.

```
# exit
$ ssh ubnt@<IP_ADDRESS>
```

### Set up a firewall

 - Make sure that the root directory is not writeable by group.

```
$ sudo chmod g-w /
```

 - We will use the built-in `ufw` firewall. You can see what apps are already recognized by `ufw` by using the following command.

```
$ sudo ufw app list
Available applications:
  OpenSSH
```

 - Here you can see that it recognizes that OpenSSH is installed. Before enabling the firewall, we need to make sure that OpenSSH is allowed so that we can remain logged in.

```
$ sudo ufw allow OpenSSH
```

 - Now we can enable the firewall.

```
$ sudo ufw enable
```

 - Next we'll verify that the firewall is working.

```
$ sudo ufw status
Status: active
 
To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
```

With the firewall enabled, the server is now reasonably secured.

## Update the system

Now is a good time to update all of the preinstalled packages on the server.

```
$ sudo apt-get update && sudo apt-get upgrade -y
```

## Install Unifi

### Install the `unifi` package from repository

 - Add the Unifi repository.

```
$ echo 'deb http://www.ubnt.com/downloads/unifi/debian stable ubiquiti' | sudo tee /etc/apt/sources.list.d/100-ubnt-unifi.list
```

 - Install the GPG key for the repository.

```
$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv 06E85760C0A52C50
```

 - Install the `unifi` package.

```
$ sudo apt-get update && sudo apt-get install unifi -y
```

 - Verify the `unifi` package is installed and running.

```
$ sudo service unifi status
```

### Create a unifi profile for `ufw` firewall

 - Create a configuration file for the profile.

```
$ sudo nano /etc/ufw/applications.d/unifi
```

 - Paste the following contents into the nano text editor.

```
[Unifi]
title=UniFi Controller
description=The UniFi Controller software is used to provision, monitor, and administrate Ubiquiti devices.
ports=8080,8443,8843,8880/tcp|3478/udp
```

 - To save the file press `Ctrl`+`x`, then type `Y` and press `Enter`.

If everything was done correctly, `ufw` will now recognize the Unifi app.

```
$ sudo ufw app list
Available applications:
  OpenSSH
  Unifi
```

 - Enable the unifi app.

```
$ sudo ufw allow Unifi
```

```
$ sudo ufw status
Status: active
 
To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
Unifi                      ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
Unifi (v6)                 ALLOW       Anywhere (v6)
```

### Run the UniFi set up wizard

The server is ready for all intents and purposes. Visit the following URL in your browser to continue setting up UniFi with supplied wizard.

    https://<IP-ADDRESS>:8443

![screenshot](screenshots/unifi11.png)

-------------------------------------------------------------------------------

#### Additional Reading

 - [How To Create Your First DigitalOcean Droplet](https://www.digitalocean.com/community/tutorials/how-to-create-your-first-digitalocean-droplet)
 - [How To Connect To Your Droplet with SSH](https://www.digitalocean.com/community/tutorials/how-to-connect-to-your-droplet-with-ssh)
 - [How To Log Into Your Droplet with PuTTY (for windows users)](https://www.digitalocean.com/community/tutorials/how-to-log-into-your-droplet-with-putty-for-windows-users)
 - [Initial Server Setup with Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04)
 - [UFW Essentials: Common Firewall Rules and Commands](https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands)
 - [UniFi - How to Install & Update via APT on Debian or Ubuntu](https://help.ubnt.com/hc/en-us/articles/220066768-UniFi-How-to-Install-Update-via-APT-on-Debian-or-Ubuntu)
 - [UniFi - Ports Used](https://help.ubnt.com/hc/en-us/articles/218506997-UniFi-Ports-Used)
