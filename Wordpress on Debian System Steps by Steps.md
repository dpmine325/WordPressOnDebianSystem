## [VPS] ALL-IN-ONE SERVER - UBUNTU 18.04

### Apps
* Apache
* PHP-FPM
* PHP7.2
* PhpRedis
* MariaDB
* Redis Server
* Composer
* letsencrypt

### STEP 1 - Create New User
1. Login to the VPS  
   >user@local:~$`ssh root@vps-ip-address`
2. Create new user account with bash access
   >root@vps:~#`useradd -m -s /bin/bash newuser`
3. Create password
   >#`passwd newuser`
4. Add new user to sudoers group
   >#`usermod -aG sudo newuser`
5. Logout from VPS

### STEP 2 - SSH With Key
1. Create ssh-key on local pc
   >user@local:~$`ssh-keygen -t rsa`
2. Copy ssh-key to VPS
   >user@local:~$`ssh-copy-id ~/.ssh/id_rsa.pub newuser@vps-ip-address`
3. Login to VPS with no password
   >user@local:~$`ssh newuser@vps-ip-address`
   
### STEP 3 - Securing SSH Access
1. SSH Configuration
   >newuser@vps:~$`sudo vim /etc/ssh/sshd_config` 
   
   Change current configuration to:  
   
   ```conf
   ChallengeResponseAuthentication no  # No need, we use Public key
   PasswordAuthentication no           # No need, we use Public key
   UsePAM no                           # No need, we use Public key
   PermitRootLogin no                  # No root login
   Protocol 2                          # Protocol 1 is older and is less secure
   ClientAliveInterval 300             # Kick idle user in 5 minutes (60*5=300 secs)
   ClientAliveCountMax 0               # Don't keep alive all idle user
   Port 1122                           # Default is 22, change to a non standard port
   ```  
   Save configuration and clos sshd_config  
2. Enable FireWall
   >newuser@vps:~$`sudo ufw enable` 
3. Allow SSH
   >newuser@vps:~$`sudo ufw allow 'OpenSSH'`
4. Allow SSH Custom Port (in case cannot ssh with new port)
   >newuser@vps:~$`sudo ufw allow 1122`
5. Reload SSH
   >newuser@vps:~$`sudo systemctl reload ssh`

### STEP 4 - Keep VPS UpToDate
>newuser@vps:~$`sudo apt update -y && sudo apt upgrade -y && sudo apt autoremove -y`

### STEP 5 Install Apache, PHP7.2, and PHP-FPM
>newuser@vps:~$`sudo apt-get -y install apache2 apache2-doc apache2-utils libapache2-mod-php php7.2 php7.2-common php7.2-gd php7.2-mysql php7.2-imap php7.2-cli php7.2-cgi libapache2-mod-fcgid apache2-suexec-pristine php-pear mcrypt imagemagick libruby libapache2-mod-python php7.2-curl php7.2-intl php7.2-pspell php7.2-recode php7.2-sqlite3 php7.2-tidy php7.2-xmlrpc php7.2-xsl memcached php-memcache php-imagick php-gettext php7.2-zip php7.2-mbstring php-soap php7.2-soap php7.2-fpm php7.2-opcache php-apcu`

### STEP 6 Apache Web Server Configuration
1. Prevent [HTTPOXY](https://httpoxy.org/) attack
   >newuser@vps:~$`sudo vim /etc/apache2/conf-available/httpoxy.conf`  
   
   Insert this configuration  
   
   ```conf
   <IfModule mod_headers.c> 
      RequestHeader unset Proxy early 
   </IfModule>
   ```  
   Save and close config file, then enable it
   >newuser@vps:~$`sudo a2enconf httpoxy`
2. Enable necesarry apache module
   >newuser@vps:~$ `sudo a2enmod suexec rewrite ssl actions include cgi dav_fs dav auth_digest headers proxy_fcgi alias`
3. Add rule firewall to allow port 80, 443
   >newuser@vps:~$`sudo ufw allow 'Apache Full'`  
   
   >newuser@vps:~$`sudo ufw delete allow 'Apache'`
4. Enable and restart Apache service
   >newuser@vps:~$`sudo systemctl enable apache2`  
   
   >newuser@vps:~$`sudo systemctl restart apache2`

### STEP 7 Enable PHP-FPM
1. Enable php-fpm module
   >newuser@vps:~$`sudo a2enconf php7.2-fpm`
2. Enable and start php-fpm service
   >newuser@vps:~$`sudo systemctl enable php7.2-fpm`  
   
   >newuser@vps:~$`sudo systemctl start php7.2-fpm`
3. Reload apache
   >newuser@vps:~$`sudo systemctl reload apache2`

### STEP 8 Configuration to use PHP-FPM
>newuser@vps:~$`sudo vim /etc/apache2/sites-enabled/000-default.conf`  
Set configuration like this:  
```conf
   <VirtualHost *:80>
           ServerName www.example.com
           ServerAdmin webmaster@localhost
           DocumentRoot /var/www/html

           #Log Path
           ErrorLog ${APACHE_LOG_DIR}/error.log
           CustomLog ${APACHE_LOG_DIR}/access.log combined

           <Directory /var/www/html>
                   AllowOverride All
           </Directory>
           <IfModule proxy_fcgi_module>
              # Enable http authorization headers
              <IfModule setenvif_module>
                      SetEnvIfNoCase ^Authorization$ "(.+)" HTTP_AUTHORIZATION=$1
              </IfModule>
              <FilesMatch ".+\.ph(ar|p|tml)$">
                      SetHandler "proxy:unix:/run/php/php7.2-fpm.sock|fcgi://localhost"
              </FilesMatch>
              <FilesMatch ".+\.phps$">
                      # Deny access to raw php sources by default
                      # To re-enable it's recommended to enable access to the files
                      # only in specific virtual host or directory
                      Require all denied
              </FilesMatch>
              # Deny access to files without filename (e.g. '.php')
              <FilesMatch "^\.ph(ar|p|ps|tml)$">
                      Require all denied
              </FilesMatch>
           </IfModule>
   </VirtualHost>
```  
Save and reload apache
>newuser@vps:~$`sudo systemctl reload apache2`

### STEP 9 Setup MariaDB
1. Install MariaDB
   >newuser@vps:~$`sudo apt install mariadb-server`
2. Remove MariaDB root password
   >newuser@vps:~$`sudo mysql -u root`
   >MariaDB [(none)]>`use mysql;`
   >MariaDB [(none)]>`update user set plugin='' where User='root';`
   >MariaDB [(none)]>`flush privileges;`
   >MariaDB [(none)]>`\q`
3. Securing MariaDB Installation (change root password)
   >newuser@vps:~$`sudo mysql_secure_installation`

### STEP 10 Install & Securing Redis
1. Install Redis Server
   >newuser@vps:~$`sudo apt install redis-server`
2. Setup redis as service
   >newuser@vps:~$`sudo vim /etc/redis/redis.conf`  
   Change **supervised no** to **supervised systemd**  
   Save and close config file
3. Restart redis
   >newuser@vps:~$`sudo systemctl restart redis`
4. Check redis status
   >newuser@vps:~$`sudo systemctl status redis`
5. Test redis connection
   >newuser@vps:~$`redis-cli`
   >127.0.0.1:6379>`ping`  
   if success the reply will be PONG
6. Binding redis to localhost only
   >newuser@vps:~$`sudo vim /etc/redis/redis.conf`  
   remove '#' on bind 127.0.0.1 ::1 
7. Setup redis password  
   Uncomment '# requirepass foobared' by removing the '#' and change foobared with your password  
   example: `requirepass y0uRn3wSecUR3red1zp455w0Rd!!`  
   Save and close config file
8. Restart redis server
   >newuser@vps:~$`sudo systemctl restart redis`

### STEP 11 Install PhpRedis
1. Check php version
   >newuser@vps:~$`php -v`
2. Install php-dev match the current version (7.2)
   >newuser@vps:~$`sudo apt install php7.2-dev`
3. Download latest php-redis
   >newuser@vps:~$`cd /tmp && wget https://github.com/phpredis/phpredis/archive/master.zip -O phpredis.zip`
4. Install Unzip
   >newuser@vps:~$`sudo apt install unzip`
5. Unpack and Install phpRedis
   >newuser@vps:~$`unzip -o /tmp/phpredis.zip && mv /tmp/phpredis-* /tmp/phpredis && cd /tmp/phpredis && phpize && ./configure && make && sudo make install`
6. Add phpRedis ext to PHP
   >newuser@vps:~$`sudo vim /etc/php/7.2/mods-available/redis.ini`  
   
   add `extension=redis.so` save and close  
   
   >newuser@vps:~$`sudo ln -s /etc/php/7.2/mods-available/redis.ini /etc/php/7.2/apache2/conf.d/redis.ini`  
   
   >newuser@vps:~$`sudo ln -s /etc/php/7.2/mods-available/redis.ini /etc/php/7.2/fpm/conf.d/redis.ini`
   
   >newuser@vps:~$`sudo ln -s /etc/php/7.2/mods-available/redis.ini /etc/php/7.2/cli/conf.d/redis.ini`  
   
   >newuser@vps:~$`sudo systemctl restart apache2` 
   
   >newuser@vps:~$`sudo systemctl restart php7.2-fpm.service`  
   
7. Check if phpRedis installed properly
   >newuser@vps:~$`php -r "if (new Redis() == true) { echo \"OK \r\n\"; }"`
   
### STEP 12 Install Composer
1. Download Composer installer
   >newuser@vps:~$`cd /tmp && php -r "copy('https://getcomposer.org/installer', '/tmp/composer-setup.php');"`
2. Go to https://composer.github.io/pubkeys.html and copy Installer Signature (SHA-384)
3. Verify composer installer
   >newuser@vps:~$`php -r "if (hash_file('SHA384', '/tmp/composer-setup.php') === 'sha_384_string') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('/tmp/composer-setup.php'); } echo PHP_EOL;"`
4. Install composer if verified
   >newuser@vps:~$`sudo php /tmp/composer-setup.php --install-dir=/usr/local/bin --filename=composer`
5. Remove Composer Setup
   >newuser@vps:~$`rm /tmp/composer-setup.php`

### STEP 13 Setup HTTPS
1. Installing certbot
   >newuser@vps:~$`sudo apt install python-certbot-apache`  
2. Obtaining SSL Certificate
   >newuser@vps:~$`sudo certbot --apache -d your_domain -d www.your_domain`  
   
   Sample output  
   ```
   Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
   -------------------------------------------------------------------------------
   1: No redirect - Make no further changes to the webserver configuration.
   2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
   new sites, or if you're confident your site works on HTTPS. You can undo this
   change by editing your web server's configuration.
   -------------------------------------------------------------------------------
   Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
   ```  
   
   Choose 2 and hit ENTER to redirect all traffic to https  
   
   ```
   Output
   IMPORTANT NOTES:
    - Congratulations! Your certificate and chain have been saved at:
      /etc/letsencrypt/live/your_domain/fullchain.pem
      Your key file has been saved at:
      /etc/letsencrypt/live/your_domain/privkey.pem
      Your cert will expire on 2018-07-23. To obtain a new or tweaked
      version of this certificate in the future, simply run certbot again
      with the "certonly" option. To non-interactively renew *all* of
      your certificates, run "certbot renew"
    - Your account credentials have been saved in your Certbot
      configuration directory at /etc/letsencrypt. You should make a
      secure backup of this folder now. This configuration directory will
      also contain certificates and private keys obtained by Certbot so
      making regular backups of this folder is ideal.
    - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
   ```  
   
  
--- 
### References
   1. https://blog.devolutions.net/2017/4/10-steps-to-secure-open-ssh
   2. https://www.webhostinghero.com/ubuntu-apache-php-fpm/
   3. https://www.ijasnahamed.in/2016/03/setup-redis-and-redis-php-client-in.html
   4. https://getcomposer.org/download/

### EOF
