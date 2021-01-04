# fusion-pbx-installation

Fusionpbx v4 Freeswitch v1.6 CentOS v7 Install Guide
Submitted by powerpbx on Sat, 01/02/2016 - 09:07
Fusionpbx

Fusionpbx is a full featured mult-tenant GUI for Freeswitch.  This guide covers the installation of Fusionpbx and Freeswitch® with MariaDB and Apache on CentOS v7. 

Tested on:
CentOS v7
Freeswitch v1.6
FusionPBX v4
MariaDB v5.5

Assumptions:
Console text mode (multi-user.target)
Installation done as root user (#)

Install Prerequisites
Ensure all required packages are installed. 

yum install epel-release
yum update
yum install git nano httpd mariadb-server php php-common php-pdo php-soap php-xml php-xmlrpc php-mysql php-cli php-imap php-mcrypt mysql-connector-odbc memcached ghostscript libtiff-devel libtiff-tools at
Disable Selinux
Check status

sestatus
If not disabled, set SELINUX=disabled in /etc/selinux/config.  Requires reboot for changes to take effect.

sed -i 's/\(^SELINUX=\).*/\SELINUX=disabled/' /etc/selinux/config
Timezone
## FIND YOUR TIMEZONE
tzselect

## SET TIMEZONE EXAMPLE
timedatectl set-timezone America/Vancouver

## CHECK TIMEZONE
​timedatectl status
Memcached
Restrict memcached to localhost to prevent it from being used for DDoS attacks.

nano /etc/sysconfig/memcached
PORT="11211"
USER="memcached"
MAXCONN="1024"
CACHESIZE="64"
OPTIONS="-l 127.0.0.1"
Reboot
To ensure the changes/additions are at their default states.

reboot
Install
Freeswitch
rpm -Uvh http://files.freeswitch.org/freeswitch-release-1-6.noarch.rpm
yum install freeswitch-config-vanilla freeswitch-sounds* freeswitch-lang* freeswitch-lua freeswitch-xml-cdr
Database
systemctl start mariadb
mysql
From mysql prompt >

CREATE DATABASE freeswitch;
GRANT ALL PRIVILEGES ON freeswitch.* TO fusionpbx@localhost IDENTIFIED BY 'somepassword';
flush privileges;
\q
ODBC
nano /etc/odbc.ini
[freeswitch]
Driver   = MySQL
SERVER   = 127.0.0.1
PORT     = 3306
DATABASE = freeswitch
OPTION  = 67108864
Socket   = /var/lib/mysql/mysql.sock
threading=0
MaxLongVarcharSize=65536

[fusionpbx]
Driver   = MySQL
SERVER   = 127.0.0.1
PORT     = 3306
DATABASE = fusionpbx
OPTION  = 67108864
Socket   = /var/lib/mysql/mysql.sock
threading=0
Test odbc driver

odbcinst -s -q
Test odbc connection

isql -v freeswitch fusionpbx somepassword 

quit
Download Fusionpbx
There are fixes and enhancements in our fork so that it will install properly on MySQL.  The developer appears to have addressed them now, so if you want the latest updates you can install from https://github.com/fusionpbx/fusionpbx

Make sure the "." is included at the end of the git clone command.  That tells git to clone into the current directory instead of creating a /fusionpbx subdirectory.

cd /var/www/html
git clone -b 4.2 https://github.com/powerpbx/fusionpbx.git .
Copy conf Directory
mv /etc/freeswitch /etc/freeswitch.orig
mkdir /etc/freeswitch
cp -R /var/www/html/resources/templates/conf/* /etc/freeswitch
Apache config
# Add user freeswitch to group apache to avoid problems with /var/lib/php/sessions directory 
usermod -a -G apache freeswitch

# Set http server to run as same user/group as Freeswitch
sed -i "s/User apache/User freeswitch/" /etc/httpd/conf/httpd.conf
sed -i "s/Group apache/Group daemon/" /etc/httpd/conf/httpd.conf

# Set webserver to obey any .htaccess files in /var/www/html and subdirs 
sed -i ':a;N;$!ba;s/AllowOverride None/AllowOverride All/2' /etc/httpd/conf/httpd.conf
Set ownership and permissions
# Ownership
chown -R freeswitch.daemon /etc/freeswitch /var/lib/freeswitch \
/var/log/freeswitch /usr/share/freeswitch /var/www/html

# Directory permissions to 770 (u=rwx,g=rwx,o='')
find /etc/freeswitch -type d -exec chmod 770 {} \;
find /var/lib/freeswitch -type d -exec chmod 770 {} \;
find /var/log/freeswitch -type d -exec chmod 770 {} \;
find /usr/share/freeswitch -type d -exec chmod 770 {} \;
find /var/www/html -type d -exec chmod 770 {} \;

# File permissions to 664 (u=rw,g=rw,o=r)
find /etc/freeswitch -type f -exec chmod 664 {} \;
find /var/lib/freeswitch -type f -exec chmod 664 {} \;
find /var/log/freeswitch -type f -exec chmod 664 {} \;
find /usr/share/freeswitch -type f -exec chmod 664 {} \;
find /var/www/html -type f -exec chmod 664 {} \;
Systemd config
nano /etc/systemd/system/freeswitch.service
[Unit]
Description=FreeSWITCH
Wants=network-online.target
After=syslog.target network-online.target
After=mariadb.service httpd.service

[Service]
Type=forking
User=freeswitch
ExecStartPre=/usr/bin/mkdir -m 0750 -p /run/freeswitch
ExecStartPre=/usr/bin/chown freeswitch:daemon /run/freeswitch
WorkingDirectory=/run/freeswitch
PIDFile=/run/freeswitch/freeswitch.pid
EnvironmentFile=-/etc/sysconfig/freeswitch
ExecStart=/usr/bin/freeswitch -ncwait -nonat $FREESWITCH_PARAMS
ExecReload=/usr/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
Create the $FREESWITCH_PARAMS file for extra parameters.  If freeswitch was installed from RPM this will probably already exist.

nano /etc/sysconfig/freeswitch
## Type:                string
## Default:             ""
## Config:              ""
## ServiceRestart:      freeswitch
#
# if not empty: parameters for freeswitch
#
FREESWITCH_PARAMS=""
Enable services
systemctl daemon-reload
systemctl enable mariadb
systemctl enable httpd
systemctl enable freeswitch
systemctl enable memcached
systemctl restart freeswitch
Fix fs_cli
If fs_cli command does not work with freeswitch running change the following config line. 

nano /etc/freeswitch/autoload_configs/event_socket.conf.xml
<param name="listen-ip" value="127.0.0.1"/>
systemctl restart freeswitch
 
Reboot and browse to the public IP address of the server
 http://xx.xx.xx.xx 

to complete the install using the following:

Username: superadmin (or whatever you want)
Password: somepassword (use whatever you want)

Database Name: fusionpbx
Database Username: fusionpbx
Database Password: somepassword
Create Database Options: check "Create the database"
Create Database Username: root
Create Database Password : (leave blank unless you have already added a root password)

Post install tasks are mandatory.

Post Install
Lock down the database server
mysql_secure_installation
systemctl restart mariadb
Answer Y to everything.

Enable freeswitch database connection
This sets Freeswitch to use mysql instead of sqlite.

nano /etc/freeswitch/autoload_configs/switch.conf.xml
<param name="core-db-dsn" value="freeswitch:fusionpbx:somepassword" /> 
systemctl restart freeswitch
Change Voicemail to Email app configuration
nano +119 /etc/freeswitch/autoload_configs/switch.conf.xml
<param name="mailer-app" value="/usr/bin/php /var/www/html/secure/v_mailto.php"/>
                <param name="mailer-app-args" value="-t"/>
systemctl restart freeswitch
Configure firewall
firewall-cmd --permanent --zone=public --add-service={http,https}
firewall-cmd --permanent --zone=public --add-port={5060,5061,5080,5081}/tcp
firewall-cmd --permanent --zone=public --add-port={5060,5061,5080,5081}/udp
firewall-cmd --permanent --zone=public --add-port=16384-32768/udp
firewall-cmd --reload
