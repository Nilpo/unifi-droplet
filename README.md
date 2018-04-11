Setting up Ubiquiti Unifi on DigitalOcean
=========================================

### Overview

Ubiquiti's UniFi controller is a great way to manage networks built on Ubiquiti infrastructure. As a service provider, you may wish to host a centralized controller in the cloud so that you can manage multiple networks from a single location. This how-to demonstrates how you might go about deploying Ubiquiti UniFi on a DigitalOcean droplet for easy scalability.

#### Contents <a name="top"></a>
+ [Prerequisites](#pr)
+ [Creating a droplet on DigitalOcean](#do)
+ [Connecting to a droplet](#cd)
+ [Setting up an Ubuntu environment](#su)
  - [Add a regular account](#su)
  - [Set up a firewall](#ufw)
  - [Update the system](#ud)
+ [Install UniFi](#un)
  - [Install the `unifi` package from repository](#un)
  - [Create a unifi profile for `ufw` firewall](#ufw2)
  - [Run the UniFi Setup Wizard](#us)
+ [Security best practices (Recommended)](#se)
  - [Disabling root login by SSH](#se)
  - [Setting up passwordless SSH login](#ps)
  - [Disabling password authentication on the server](#pa)
  - [Install Fail2Ban](#f2b)
  - [Force a password when using `sudo`](#sudo)
+ [Additional Reading](#ar)


### Prerequisites <a name="pr"></a>

* A DigitalOcean account
* A UniFi cloud account (optional)

[back to top](#top)

___

### Creating a droplet on DigitalOcean <a name="do"></a>

 1. After logging in to the DigitalOcean [dashboard][1], click the **Create** button and choose **Droplets** from the expanding menu.

    <p align="center"><img src="screenshots/unifi01.png" width="50%" height="50%"></p>

 1. Under **Choose an image**, select **One-click apps** and then choose **MongoDB 3.4.10 on 16.04**.

    <p align="center"><img src="screenshots/unifi02.png" width="50%" height="50%"></p>

 1. Under **Choose a size**, select a size that fits your budget. (The smallest size should be just fine to get started. It's easy to scale your application up as you grow.)

    <p align="center"><img src="screenshots/unifi03.png" width="50%" height="50%"></p>

 1. Under **Choose a datacenter region**, select a region that is close to you and your clients.
 1. Under **Select additional options**, select both `Backups` and `Monitoring`. These free addons are important for any production system.
 1. Under **Add your SSH keys**, select the SSH key for the machine you are currently working on or create one. If you choose not to use SSH authentication, you can still log in to your droplet using a username and password but SSH is the preferred method because of its increased security. More information about SSH logins can be found [here][4].

    <p align="center"><img src="screenshots/unifi04.png" width="50%" height="50%"></p>

 1. Under **Finalize and create**, assign your droplet and appropriate name and click the `Create` button.

    <p align="center"><img src="screenshots/unifi05.png" width="50%" height="50%"></p>

Once your droplet is created, its details can be viewed in your droplets list. Make a note of the IP address that has been assigned to your droplet. You'll need this later along with the temporary password that you will receive in your welcome email.

<p align="center"><img src="screenshots/unifi06.png" width="50%" height="50%"></p>

[back to top](#top)

### Connecting to a droplet <a name="cd"></a>

You will use SSH to connect to your new DigitalOcean droplet. For Linux and Mac systems, this is easiest done using OpenSSH at the command line. Windows users are recommended to use the free [PuTTY][2] tool.

Fill in your details on the PuTTY Configuration dialog. Pay special attention the the IP address. Make sure that it matches the IP address of your new droplet.

<p align="center"><img src="screenshots/unifi07.png" width="50%" height="50%"></p>

Log in using the username *root* and the password that was supplied to you. When logging in for the first time, you will be required to set a permanent password for your root account. Once you have done this, you will be logged in to your droplet and you will have a root prompt.

<p align="center"><img src="screenshots/unifi09.png" width="50%" height="50%"></p>

PuTTY will also warn you that it does not recognize the certificate for the server your are connecting to. Since this is your first time connecting, select `Yes` to accept the server's certificate and open the connection.

<p align="center"><img src="screenshots/unifi08.png" width="50%" height="50%"></p>

To connect from the command line instead, use the following command:

	ssh root@<IP_ADDRESS>

<p align="center"><img src="screenshots/unifi10.png" width="50%" height="50%"></p>

[back to top](#top)

### Setting up an Ubuntu environment <a name="su"></a>

#### Add a regular account

 1. Add a regular user account.

    ```shell
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

 1. Add the new user account to the sudoers group so that you can perform privileged commands.

    ```shell
    # usermod -aG sudo ubnt
    ```

 1. (Optional) If you chose to add an SSH key during droplet creation, you'll need to copy the authorized key to your new user before logging in. If you did not choose to set up SSH access during creation, you can skip this step.
 
   ```shell
   # mkdir -p /home/ubnt/.ssh
   # cp /root/.ssh/authorized_keys /home/ubnt/.ssh/authorized_keys
   # chmod 700 /home/ubnt/.ssh
   # chmod 644 /home/ubnt/.ssh/authorized_keys
   # chown -R ubnt:ubnt /home/ubnt/

 1. Log out and log back in using the newly created user.

    ```shell
    # exit
    $ ssh ubnt@<IP_ADDRESS>
    ```

[back to top](#top)

#### Set up a firewall <a name="ufw"></a>

 1. Make sure that the root directory is not writeable by group.

    ```shell
    $ sudo chmod g-w /
    ```

 1. We will use the built-in `ufw` firewall. You can see what apps are already recognized by `ufw` by using the following command.

    ```shell
    $ sudo ufw app list
    Available applications:
      OpenSSH
    ```

 1. Here you can see that it recognizes that OpenSSH is installed. Before enabling the firewall, we need to make sure that OpenSSH is allowed so that we can remain logged in.

    ```shell
    $ sudo ufw allow OpenSSH
    ```

 1. Now we can enable the firewall.

    ```shell
    $ sudo ufw enable
    ```

 1. Next we'll verify that the firewall is working.

    ```shell
    $ sudo ufw status
    Status: active
     
    To                         Action      From
    --                         ------      ----
    OpenSSH                    ALLOW       Anywhere
    OpenSSH (v6)               ALLOW       Anywhere (v6)
    ```

With the firewall enabled, the server is now reasonably secured.

[back to top](#top)

#### Update the system <a name="ud"></a>

Now is a good time to update all of the preinstalled packages on the server.

```shell
$ sudo apt-get update && sudo apt-get upgrade -y
```

[back to top](#top)

### Install UniFi <a name="un"></a>

#### Install the `unifi` package from repository

 1. Add the Unifi repository.

    ```shell
    $ echo 'deb http://www.ubnt.com/downloads/unifi/debian stable ubiquiti' | sudo tee /etc/apt/sources.list.d/100-ubnt-unifi.list
    ```

 1. Install the GPG key for the repository.

    ```shell
    $ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv 06E85760C0A52C50
    ```

 1. Install the `unifi` package.

    ```shell
    $ sudo apt-get update && sudo apt-get install unifi -y
    ```

 1. Verify the `unifi` package is installed and running.

    ```shell
    $ sudo service unifi status
    ```

[back to top](#top)

#### Create a unifi profile for `ufw` firewall <a name="ufw2"></a>

 1. Create a configuration file for the profile.

    ```shell
    $ sudo nano /etc/ufw/applications.d/unifi
    ```

 1. Paste the following contents into the nano text editor.

    ```
    [Unifi]
    title=UniFi Controller
    description=The UniFi Controller software is used to provision, monitor, and administrate Ubiquiti devices.
    ports=8080,8443,8843,8880/tcp|3478/udp
    ```

 1. To save the file press `Ctrl`+`x`, then type `Y` and press `Enter`.

    If everything was done correctly, `ufw` will now recognize the Unifi app.

    ```shell
    $ sudo ufw app list
    Available applications:
      OpenSSH
      Unifi
    ```

 1. Enable the unifi app.

    ```shell
    $ sudo ufw allow Unifi
    ```

    ```shell
    $ sudo ufw status
    Status: active
     
    To                         Action      From
    --                         ------      ----
    OpenSSH                    ALLOW       Anywhere
    Unifi                      ALLOW       Anywhere
    OpenSSH (v6)               ALLOW       Anywhere (v6)
    Unifi (v6)                 ALLOW       Anywhere (v6)
    ```

[back to top](#top)

#### Run the UniFi Setup Wizard <a name="us"></a>

The server is ready for all intents and purposes. Visit the following URL in your browser to continue setting up UniFi with supplied wizard.

    https://<IP-ADDRESS>:8443

<p align="center"><img src="screenshots/unifi11.png" width="50%" height="50%"></p>

[back to top](#top)


### Security best practices (Recommended) <a name="se"></a>

#### Disable root login by SSH

After you [create a normal user](#su), you can disable SSH logins for the root account. This greatly improves security by eliminating the most commonly attacked account from remote logins.

 1. Log in to the server as *root* using SSH.

    ```shell
    ssh root@<IP_ADDRESS>
    ```

 1. Open the `/etc/ssh/sshd_config` file in your preferred text editor (nano, vi, etc.).

    ```shell
    $ nano /etc/ssh/sshd_config
    ```

 1. Locate the following line:

    ```
    PermitRootLogin yes
    ```

 1. Modify the line as follows:

    ```
    PermitRootLogin no
    ```

 1. Add the following line. Replace *username* with the name of the user you created earlier.

    ```
    AllowUsers username
    ```

    > This step is crucial. If you do not add the user to the list of allowed SSH users, you will be unable to log in to your server!

 1. Save the changes to the `/etc/ssh/sshd_config` file, and then exit the text editor.

 1. Restart the SSH service using the appropriate command for your Linux distribution:

    ```shell
    $ service ssh restart
    ```

 1. __While still logged in as root__, try to log in as the new user using SSH in a new terminal window. You should be able to log in. If the login fails, check your settings. __Do not exit your open root session__ until you are able to log in as the normal user in another window.

[back to top](#top)

#### Set up passwordless SSH login <a name="ps"></a>

Windows users using PuTTY should follow the instructions found [here][12].

 1. Create the RSA key pair (on the local computer)

    ```shell
    $ ssh-keygen -t rsa
    Enter file in which to save the key (/username/.ssh/id_rsa): <Enter>
    Enter passphrase (empty for no passphrase): 
    ```

 1. Copy the new public key to the server using SSH. Be sure to change *username* and *IP_ADDRESS* to match your server.

    ```shell
    $ cat ~/.ssh/id_rsa.pub | ssh username@IP_ADDRESS "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
    ```

#### Disable password authentication on the server <a name="pa"></a>

 1. Log in to your server with your SSH key for the first time. From the terminal, this done the exact same way with the exception that you will not be prompted for a password.

    ```shell
    ssh username@<IP-ADDRESS>
    ```

 1. Edit the `/etc/ssh/shsd_config` file.

    ```shell
    $ sudo nano /etc/ssh/sshd_config
    ```

 1. Change the `PasswordAuthentication` directive value to `no`.

    ```
    PasswordAuthentication no
    ```

 1. Restart SSH.

    ```shell
    $ sudo service ssh restart
    ```

#### Install Fail2Ban <a name="f2b"></a>

Leaving the SSH port open to the public, as we've done, presents a potential risk. Anyone can attempt a connection to your droplet from anywhere. While using `ufw` to lock the SSH port down to connections from whitelisted IP addresses is a great boost to security, it comes at the cost of usability since you also won't be able to access your droplet except from whitelisted IP addresses. If you ever need to access your UniFi controller from the field, that would present a problem for you. So rather than locking down the port itself, a service such as `fail2ban` can help to protect your droplet. The Fail2Ban service monitors access logs for suspicious activity and proactively sets firewall rules based upon limitations that you provide. As an example, Fail2Ban can implement flood control that bans an specific IP address after a specified number of invalid login attempts. This is especially useful for mitigating brute force attacks.

 1. Install Fail2Ban.

    ```shell
    $ sudo apt-get update
    $ sudo apt-get install fail2ban
    ```

On Ubuntu, Fail2Ban's defaults will protect SSH by blocking IP addresses if there are three failed login attempts within 10 minutes. These options are configurable.

[back to top](#top)

#### Force a password when using `sudo` <a name="sudo"></a>

Forcing sudo to require a password is a form of two-factor authentication. In the event that your private SSH key is ever compromised, it will limit an attacker's ability to damage your system by preventing the use of any commands that require sudo by prompting for a password every time. By default, this is not the case and normal users with sudo permission can simply execute commands as root without any additional authentication.

 1. Edit `/etc/sudoers.d/90-cloud-init-users`.

    ```shell
    $ sudo nano /etc/sudoers.d/90-cloud-init-users
    ```

 1. Edit the existing line for root and add one for the the new user. Make sure that `NOPASSWD` is changed to `PASSWD`.

    ```shell
    # User rules for root
    root ALL=(ALL) PASSWD:ALL
    ubnt ALL=(ALL) PASSWD:ALL
    ```

 1. Save the changes and exit Nano.

[back to top](#top)

___

#### Additional Reading <a name="ar"></a>

 + [How To Create Your First DigitalOcean Droplet][3]
 + [How To Connect To Your Droplet with SSH][4]
 + [How To Log Into Your Droplet with PuTTY (for windows users)][5]
 + [Initial Server Setup with Ubuntu 16.04][6]
 + [UFW Essentials: Common Firewall Rules and Commands][7]
 + [UniFi - How to Install & Update via APT on Debian or Ubuntu][8]
 + [UniFi - Ports Used][9]
 + [How To Configure SSH Key-Based Authentication on a Linux Server][10]
 + [How To Use SSH Keys with DigitalOcean Droplets][11]
 + [How To Use SSH Keys with PuTTY on DigitalOcean Droplets (Windows users)][12]
 + [How To Protect SSH with Fail2Ban on Ubuntu 14.04][13]

[1]: https://cloud.digitalocean.com/droplets
[2]: http://www.putty.org/
[3]: https://www.digitalocean.com/community/tutorials/how-to-create-your-first-digitalocean-droplet
[4]: https://www.digitalocean.com/community/tutorials/how-to-connect-to-your-droplet-with-ssh
[5]: https://www.digitalocean.com/community/tutorials/how-to-log-into-your-droplet-with-putty-for-windows-users
[6]: https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04
[7]:https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands
[8]: https://help.ubnt.com/hc/en-us/articles/220066768-UniFi-How-to-Install-Update-via-APT-on-Debian-or-Ubuntu
[9]: https://help.ubnt.com/hc/en-us/articles/218506997-UniFi-Ports-Used
[10]: https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server
[11]: https://www.digitalocean.com/community/tutorials/how-to-use-ssh-keys-with-digitalocean-droplets
[12]: https://www.digitalocean.com/community/tutorials/how-to-use-ssh-keys-with-putty-on-digitalocean-droplets-windows-users
[13]: https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04
