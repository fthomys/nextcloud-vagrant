Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"  
  config.vm.hostname = "nextcloud-vagrant"
  config.vm.network "private_network", type: "dhcp"

  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "forwarded_port", guest: 443, host: 8443
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "4096" 
    vb.cpus = 4 
  end

  config.vm.provision "shell", inline: <<-SHELL
    export DEBIAN_FRONTEND=noninteractive

    echo "Updating system packages..."
    apt update && apt upgrade -y

    echo "Installing required packages..."
    apt install -y apache2 mariadb-server libapache2-mod-php php php-gd php-mysql php-curl php-mbstring \
                   php-intl php-bcmath php-imagick php-xml php-zip unzip curl wget sudo ufw fail2ban \
                   php-apcu php-igbinary php-redis redis-server php-cli php-common php-opcache

    echo "Configuring MariaDB security..."
    randomPass=$(openssl rand -base64 16)
    mysql_secure_installation <<EOF
    n
    y
    y
    y
    y
EOF

    echo "Creating Nextcloud database and user..."
    mysql -uroot -e "CREATE DATABASE nextcloud;"
    mysql -uroot -e "CREATE USER 'nextcloud'@'localhost' IDENTIFIED BY '${randomPass}';"
    mysql -uroot -e "GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost';"
    mysql -uroot -e "FLUSH PRIVILEGES;"

    echo "Enabling Apache modules and configuring virtual host..."
    a2enmod rewrite headers env dir mime setenvif ssl proxy_fcgi setenvif

    echo "Generating SSL certificate..."
    mkdir -p /etc/apache2/ssl
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/apache2/ssl/nextcloud.key -out /etc/apache2/ssl/nextcloud.crt \
      -subj "/C=DE/ST=Berlin/L=Berlin/O=Nextcloud/OU=VagrantTest/CN=nextcloud-vagrant"

    cat <<EOF > /etc/apache2/sites-available/nextcloud.conf
<VirtualHost *:80>
    ServerName nextcloud-vagrant
    Redirect permanent / https://nextcloud-vagrant/
</VirtualHost>

<VirtualHost *:443>
    ServerAdmin admin@nextcloud.local
    DocumentRoot /var/www/html
    ServerName nextcloud-vagrant

    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/nextcloud.crt
    SSLCertificateKeyFile /etc/apache2/ssl/nextcloud.key

    <Directory /var/www/html/>
        Options FollowSymlinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog \${APACHE_LOG_DIR}/error.log
    CustomLog \${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
EOF

    a2ensite nextcloud.conf
    systemctl restart apache2

    echo "Downloading and installing Nextcloud..."
    cd /var/www
    wget https://download.nextcloud.com/server/releases/latest.tar.bz2
    tar -xjf latest.tar.bz2
    chown -R www-data:www-data nextcloud
    chmod -R 755 nextcloud

    rm -rf html
    mv nextcloud html

    echo "Configuring Nextcloud..."
    sudo -u www-data php /var/www/html/occ maintenance:install \
        --database "mysql" --database-name "nextcloud" \
        --database-user "nextcloud" --database-pass "${randomPass}" \
        --admin-user "admin" --admin-pass "${randomPass}"

    sudo -u www-data php /var/www/html/occ config:system:set trusted_domains 0 --value="*"
    sudo -u www-data php /var/www/html/occ config:system:set trusted_domains 1 --value=localhost
    sudo -u www-data php /var/www/html/occ config:system:set trusted_domains 2 --value=nextcloud-vagrant

    echo "Setting up Redis cache for Nextcloud..."
    sudo -u www-data php /var/www/html/occ config:system:set memcache.local --value='\\OC\\Memcache\\APCu'
    sudo -u www-data php /var/www/html/occ config:system:set memcache.locking --value='\\OC\\Memcache\\Redis'
    sudo -u www-data php /var/www/html/occ config:system:set memcache.distributed --value='\\OC\\Memcache\\Redis'

    sudo -u www-data php /var/www/html/occ config:system:set redis host --value='localhost'
    sudo -u www-data php /var/www/html/occ config:system:set redis port --value='6379'


    echo "Configuring PHP performance optimizations..."
    cat <<EOF > /etc/php/8.2/apache2/conf.d/90-nextcloud.ini
    opcache.enable=1
    opcache.interned_strings_buffer=16
    opcache.max_accelerated_files=10000
    opcache.memory_consumption=128
    opcache.save_comments=1
    opcache.revalidate_freq=60
EOF

    systemctl restart apache2

    echo "Enabling firewall rules..."
    ufw allow 80
    ufw allow 443
    ufw allow 22
    yes | ufw enable

    echo "Setting up Fail2Ban to protect against attacks..."
    cat <<EOF > /etc/fail2ban/jail.local
[nextcloud]
enabled = true
port = http,https
filter = nextcloud
logpath = /var/www/html/data/nextcloud.log
maxretry = 5
EOF
    systemctl restart fail2ban

    systemctl restart redis-server apache2
    sudo -u www-data php /var/www/html/occ maintenance:repair


    echo "Creating a swap file..."
    fallocate -l 2G /swapfile
    chmod 600 /swapfile
    mkswap /swapfile
    swapon /swapfile
    echo '/swapfile none swap sw 0 0' | tee -a /etc/fstab

    echo "Setting up systemd service for Nextcloud background jobs..."
    cat <<EOF > /etc/systemd/system/nextcloud-cron.service
[Unit]
Description=Nextcloud cron job
After=network.target

[Service]
User=www-data
ExecStart=/usr/bin/php -f /var/www/html/cron.php

[Install]
WantedBy=multi-user.target
EOF
    systemctl enable --now nextcloud-cron.service

    echo "Writing credentials to /var/nclogin.txt..."
    cat <<EOF > /var/nclogin.txt
Nextcloud login data:
Username: admin
Password: ${randomPass}
Database: nextcloud
Database user: nextcloud
Database password: ${randomPass}
EOF
    chmod 600 /var/nclogin.txt

    echo "Nextcloud is ready at https://localhost (login: admin | ${randomPass})"
  SHELL
end

