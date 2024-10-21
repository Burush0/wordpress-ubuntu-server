# wordpress-ubuntu-server
Deploying Wordpress on an Ubuntu server, running on an Apache2 server behind an Nginx reverse proxy. (As of October 2024)

# Prerequisites:
- An [Ubuntu Server](https://ubuntu.com/download/server) installation, preferably fresh. I did mine inside a VMWare Workstation virtual machine.

  Follow the setup, it's not that difficult, I will not be going in detail on how to install an OS here.
- A way to generate SSH keys on your host machine. In my case of Windows 10, I had [Git](https://gitforwindows.org/) installed, that can also be achieved with [PuTTY](https://www.putty.org/).

# Setup:
## 1. Ensure all the software packages on the Ubuntu Server are up to date:
```
sudo apt update -y && sudo apt upgrade -y
```
## 2. Setup SSH connection between guest (VM) and host (your PC):

I find it very convenient to be able to connect to the VM through SSH, as it allows me to copy-paste commands with ease. In a real-world scenario, you will most likely be accessing remote servers with SSH, so this is good practice.

### 2.1. Install OpenSSH Server:

```
sudo apt install openssh-server -y
```

### 2.2. Generate an SSH key on your host machine:
```
ssh-keygen -t rsa -b 4096
```
- -t stands for type. RSA is the default type.
 
- -b stands for bits. By default the key is 3072 bits long. 4096 bits = stronger security.

You will be prompted to come up with a passphrase, it's optional but recommended.

Once you do that, a public-private keypair will be generated in .ssh in your user directory.

### 2.3. Send the public part of the key to the guest. 

The easiest way to do this will be to use `ssh-copy-id`:
```
ssh-copy-id <remote-user>@<server-ip>
```
The variables in this command will depend on the guest installation, your remote-user should match the username that you specified during the installation, and the server IP can be found by running the command `ip address` on the guest machine (at least the local IP, for remote SSH connections you will likely be given the machine's IP (duh))

Also, just in case, if a variable is specified like `<x>`, if you pass it in, you write it without the angle brackets. E.g. `x = 123, y = 456, <x>@<y>` becomes `123@456`. 

After you enter the `ssh-copy-id` command, you will be prompted to enter the password of your guest machine's user. Not the SSH passphrase! After you enter it, the public part of the key will be copied and you should be able to SSH over to guest now using
```
ssh <remote-user>@<server-ip>
```
This time, it will ask for the SSH passphrase and not the user password. You should be able to connect to the server now.

### 2.4. (Optional, but highly recommended) Disable password authentication on guest machine.

At first, I didn't quite understand what difference this step makes. Essentially, if you leave the password authentication enabled, and an unauthorized party gets access to your remote name and server-ip, they could, in theory, bruteforce the password to get access to the server. However, if you disable password authentication, the one and only way to remotely connect to the server will be through a keypair, making it a lot more secure.

Edit the `/etc/ssh/sshd_config` file. By default, the line `#PasswordAuthentication yes` is commented out, denoted by the \# symbol, so we have to change that to `PasswordAuthentication no`. Save changes to the file. Restart the SSH service with the following command: 
```
sudo systemctl restart ssh
```

I am writing under the assumption that you know how to edit files on Linux, but in case that bit of information is needed, I use the nano package and save my changes with `Ctrl+X, Y, Enter`. In the case above, the file is protected, so you need to run `sudo nano /etc/ssh/sshd_config` in order to be able to edit it. To all the vim wizards, I apologize that your eyes had to gaze on this paragraph of text.

Finally, we're done with the SSH part and can proceed to deploy Wordpress.

## 3. PostgreSQL - the database

By default, Wordpress works only with MySQL and MariaDB out of the box. As a part of the challenge, I decided to install PostgreSQL instead. For that, some additional steps are needed to be taken.

### 3.1. Install PostgreSQL
```
sudo apt install postgresql
```

### 3.2. Setup a database and a user for Wordpress
```
sudo -u postgres psql
postgres=# create database wordpress;
postgres=# create user wordpress with encrypted password '<YOUR_PASSWORD>';
postgres=# grant all privileges on database wordpress to wordpress;
postgres=# exit
```
#### Do not copy the `postgres=#` part, I use it to denote that we're in a psql terminal.

We enter the PostgreSQL terminal, create a database named `wordpress`, we then create a user with the name `wordpress`, we specify a password, we grant all database privileges to the newly created user. Last line is a little hard to read because both the database and the user are named wordpress, if you decide to name them something else, the syntax is as follows: 
```
grant all privilegs on database db_name to username;
```

## 4. Wordpress on Apache2

For this part, I have followed the official [Ubuntu tutorial for installing and configuring Wordpress](https://ubuntu.com/tutorials/install-and-configure-wordpress), with some adjustments. I will not go into detail what commands do and why you enter them, if you're interested, go read over there.

### 4.1. Install dependencies

The difference between the guide and the ones I have is that we do not need to install MySQL server, but we need an extra dependency `php-pgsql`.
```
sudo apt install apache2 \
                 ghostscript \
                 libapache2-mod-php \
                 php \
                 php-bcmath \
                 php-curl \
                 php-imagick \
                 php-intl \
                 php-json \
                 php-mbstring \
                 php-mysql \
                 php-pgsql \
                 php-xml \
                 php-zip
```

### 4.2. Get Wordpress

```
sudo mkdir -p /srv/www
sudo chown www-data: /srv/www
curl https://wordpress.org/latest.tar.gz | sudo -u www-data tar zx -C /srv/www
```

### 4.3. Set up Apache2 Virtual Host and ports

I plan to run my Apache2 server on port 8080, so for that, I need to edit the default configuration.

In the file `/etc/apache2/ports.conf`, on the line where it says `Listen 80`, change it to `Listen 8080` and save changes.

Create a new virtual host file at `/etc/apache2/sites-available/wordpress.conf` and fill in the following information:
```
<VirtualHost *:8080>
    DocumentRoot /srv/www/wordpress
    <Directory /srv/www/wordpress>
        Options FollowSymLinks
        AllowOverride Limit Options FileInfo
        DirectoryIndex index.php
        Require all granted
    </Directory>
    <Directory /srv/www/wordpress/wp-content>
        Options FollowSymLinks
        Require all granted
    </Directory>
</VirtualHost>
```
A slightly different version is available as a part of the repository, we later change `DocumentRoot /srv/www/wordpress` to `DocumentRoot /srv/www/` due to the way Nginx reverse proxy works.

After saving this `.conf` file, we run some commands (as per the linked guide above):
```
sudo a2ensite wordpress
sudo a2enmod rewrite
sudo a2dissite 000-default
sudo systemctl restart apache2
```

### 4.4. [PG4WP](https://github.com/PostgreSQL-For-Wordpress/postgresql-for-wordpress) (PostgreSQL for Wordpress)

```
cd /srv/www/wordpress/wp-content
sudo git clone https://github.com/PostgreSQL-For-Wordpress/postgresql-for-wordpress.git
sudo mv postgresql-for-wordpress/pg4wp pg4wp
sudo cp pg4wp/db.php db.php
sudo rm -rf postgresql-for-wordpress
```
Enter Wordpress directory;

Clone the repo;

Move the pg4wp folder over into the wp-content folder

Copy the db.php file over into the wp-content folder

Clean up the cloned repo folder which is no longer needed.

You can optionally enter the command `sudo cat /srv/www/wordpress/wp-content/db.php | grep DB_DRIVER` in order to check if the `DB_DRIVER` variable is indeed `pgsql`. It should be.

### 4.5. Set up Wordpress config to connect to PostgreSQL

```
sudo -u www-data cp /srv/www/wordpress/wp-config-sample.php /srv/www/wordpress/wp-config.php
sudo -u www-data sed -i 's/database_name_here/wordpress/' /srv/www/wordpress/wp-config.php
sudo -u www-data sed -i 's/username_here/wordpress/' /srv/www/wordpress/wp-config.php
sudo -u www-data sed -i 's/password_here/<YOUR_PASSWORD>/' /srv/www/wordpress/wp-config.php
```
You should also setup [salt](https://api.wordpress.org/secret-key/1.1/salt/) for security purposes, replace the lines in the `/srv/www/wordpress/wp-config.php` with the ones the website generates.

And with that done, you should be able to access the Wordpress deployment from the host machine's web browser by going to `<server-ip>:8080/`. If you see a Wordpress page where you are prompted to pick a language, well done! 

#### (DON'T SET IT UP YET!)

## 5. Nginx / ok im tired of typing for now let's do a half-commit so that I don't lose this and get back to it in a bit
