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

At first, I didn't quite understand what difference this step makes. Essentially, if you leave the password authentication enabled, and an unauthorized party gets access to your remote name and server-ip, they could, in theory, bruteforce the password to get access to the server. However, if you disable password authentication, the one and only way to remotely connect to the server will be through an SSH keypair, making it a lot more secure.

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

## 5. Nginx

### 5.1. Install Nginx
```
sudo apt install nginx
```

### 5.2. SSL Certificates (mostly adopted from [this](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-nginx-in-ubuntu#step-1-creating-the-tls-certificate) tutorial)

```
sudo openssl req -x509 -noenc -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
```
In interactive mode, type in information about yourself and your organization. You can leave the fields blank by typing `.` instead. This will generate a self-signed key and certificate pair.
```
sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096
```
This creates a "strong Diffie-Hellman (DH) group", whatever that means, it's needed. For more information check the link provided above.

Create a snippet file at `/etc/nginx/snippets/self-signed.conf` and put in the following 2 lines:
```
ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
```
You also need to create a configuration file for SSL at `/etc/nginx/snippets/ssl-params.conf`, this is adopted from [Cipherlist.eu](https://cipherlist.eu/)
```
ssl_protocols TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_dhparam /etc/nginx/dhparam.pem; 
ssl_ciphers EECDH+AESGCM:EDH+AESGCM;
ssl_ecdh_curve secp384r1;
ssl_session_timeout  10m;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 1.1.1.1 1.0.0.1 valid=300s;
resolver_timeout 5s;
# Disable strict transport security for now. You can uncomment the following
# line if you understand the implications.
#add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";
```
I use CloudFlare DNS instead of the Google one, just a personal preference. If you want to use Google's DNS, replace the line `resolver 1.1.1.1...` with `resolver 8.8.8.8 8.8.4.4 valid=300s;`.

### 5.3. (Optional) Specify a local domain name

I did this step so that instead of typing in the server ip in the web browser, I could instead write a domain, which looks more like a real website. You can skip this step, but it will come in handy in the future step where I specify the Nginx configuration file and use this local domain name as a server name.

In `/etc/hosts` file on your system (For Windows, that would be `C:/Windows/System32/drivers/etc/hosts`), add in a line that looks like this:
```
<server-ip> <domain-name>
```
To give a more specific example, my local server ip was `192.168.139.132` and the domain name I chose was `testdomain.com`, so the line looked like:
```
192.168.139.132 testdomain.com
```
Save changes to the file, needs administrator privileges (which means you might need to run Notepad as admin), and now you can access the Apache2 server at `http://testdomain.com:8080/` instead of the `<server-ip>:8080/`.

### 5.4. Nginx configuration

Create a configuration file at `/etc/nginx/sites-available/wordpress`:
```
server {
        listen 443 ssl;
        listen [::]:443 ssl;
        include snippets/self-signed.conf;
        include snippets/ssl-params.conf;

        root /var/www/html;

        index index.nginx-debian.html;

        server_name testdomain.com;

        location / {
                try_files $uri $uri/ =404;
        }

        location /wordpress/ {
                proxy_pass http://127.0.0.1:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Host $server_name;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_redirect off;
        }
}

server {
        listen 80;
        listen [::]:80;

        server_name testdomain.com;

        return 301 https://$server_name$request_uri;
}
```
This is quite a hefty configuration file, let's go through it:
- listens on port 443 (default HTTPS port)
- includes both snippet files, so the keypair and the SSL configuration
- the root folder for the Nginx server is `/var/www/html`
- the default page it displays is the `index.nginx-debian.html` file (auto-generated when installing Nginx)
- the server name is the one we specified in previous step, if you skipped it, type in the server IP instead
- at location `/`, we load the aforementioned index file which tells us that the Nginx server is working
- at location `/wordpress/` (note the trailing backslash, important for redirects), Nginx acts as a reverse proxy (for more information, check this piece of [documentation](https://developer.wordpress.org/advanced-administration/security/https/#using-a-reverse-proxy))
- the second server listens at port 80 (default HTTP port) and redirects all traffic to HTTPS instead

With all of this out of the way, the only thing left to do is to add a symbolic link to the `sites-enabled` folder and restart the Nginx service.
```
sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

### 5.5. Additional changes

Before we are able to access Wordpress through Nginx, some adjustments need to be done to the configuration files.

In `/srv/www/wordpress/wp-config.php`, the following lines need to be added for HTTPS traffic to work through the reverse proxy (same source as the previous part)
```
define( 'FORCE_SSL_ADMIN', true );
// in some setups HTTP_X_FORWARDED_PROTO might contain 
// a comma-separated list e.g. http,https
// so check for https existence
if( strpos( $_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false )
    $_SERVER['HTTPS'] = 'on';
```
In the Apache2 virtual host configuration file at `/etc/apache2/sites-available/wordpress.conf`, the DocumentRoot needs to be changed now:
```
...
DocumentRoot /srv/www/
...
```
Restart the Apache2 service:
```
sudo systemctl restart apache2
```
And finally, we should be able to access Wordpress via the `https://testdomain.com/wordpress/` in our web browser on the host machine. We can proceed with the Wordpress installation now.

## 6. Firewall

With the Apache2 configuration changed, if we try to access the port 8080 directly, we will not be able to reach Wordpress, since it now only works through Nginx reverse proxy. So, the best course of action would be to close port 8080. Or rather, to only have the ports 22 (SSH), 80 (HTTP) and 443 (HTTPS) be open to public. That is done with only 2 commands:
```
sudo ufw allow 22,80,443/tcp
sudo ufw --force enable
```
Congratulations! You now have a working Wordpress installation on Ubuntu Server, running on Apache2 behind an Nginx reverse proxy. Pat yourself on the back for making it that far.
