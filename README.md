# openstack.ubuntu.24.epoxy-howto

Download Ubuntu 24.04 LTS Server iso here: https://ubuntu.com/download/server

Install basic Ubuntu Server with ssh

enable root ssh login (sudo nano /etc/ssh/sshd_config) via PermitRootLogin yes

restart sshd

set root password: sudo passwd root

set NTP Server: nano /etc/systemd/timesyncd.conf by adding NTP=whatever.com

systemctl restart systemd-timesyncd

timedatectl timesync-status

apt -y install mariadb-server

nano /etc/mysql/mariadb.conf.d/50-server.cnf

\# line 95 : confirm default charset
\# if use 4 bytes UTF-8, specify [utf8mb4]
character-set-server  = utf8mb4
collation-server      = utf8mb4_general_ci

systemctl restart mariadb

mysql_secure_installation

Switch to unix_socket authentication [Y/n] n
Change the root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y

apt -y install software-properties-common

add-apt-repository cloud-archive:epoxy

apt update

apt -y upgrade

apt -y install rabbitmq-server memcached python3-pymysql nginx libnginx-mod-stream

mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.org

nano /etc/netplan/01-netcfg.yaml

network:
  ethernets:
    # interface name
    ens192:
      dhcp4: false
      # IP address/subnet mask
      addresses: [192.168.200.165/24]
      # default gateway
      # [metric] : set priority (specify it if multiple NICs are set)
      # lower value is higher priority
      routes:
        - to: default
          via: 192.168.200.1
          metric: 100
      nameservers:
        # name server to bind
        addresses: [192.168.200.1]
        # DNS search base
        search: [starfleet.local]
      dhcp6: false
    ens224:
      dhcp4: false
      # IP address/subnet mask
      addresses: [192.168.19.123/24]
      # default gateway
      # [metric] : set priority (specify it if multiple NICs are set)
      # lower value is higher priority
      routes:
        - to: default
          via: 192.168.19.1
          metric: 101
      nameservers:
        # name server to bind
        addresses: [192.168.19.3]
        # DNS search base
        search: [outside]
      dhcp6: false
  version: 2

chmod 600 /etc/netplan/01-netcfg.yaml

netplan apply

echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf

sysctl -p

ip address show

rabbitmqctl add_user openstack password

rabbitmqctl set_permissions openstack ".*" ".*" ".*"

nano /etc/mysql/mariadb.conf.d/50-server.cnf

uncomment: 
skip-name-resolve
max_connections        = 1000

nano /etc/memcached.conf
comment the ipv6 address out

unlink /etc/nginx/sites-enabled/default

systemctl restart mariadb rabbitmq-server memcached nginx

mysql
create database keystone;
grant all privileges on keystone.* to keystone@'localhost' identified by 'password';
grant all privileges on keystone.* to keystone@'%' identified by 'password';
exit

apt -y install keystone python3-openstackclient apache2 libapache2-mod-wsgi-py3 python3-oauth2client

nano /etc/keystone/keystone.conf

\# line 462 : add to specify Memcache Server
memcache_servers = 127.0.0.1:11211
\# line 711 : change to MariaDB connection info
connection = mysql+pymysql://keystone:password@127.0.0.1:3306/keystone
\# line 2493 : uncomment
provider = fernet

su -s /bin/bash keystone -c "keystone-manage db_sync"

keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone

keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

export controller=ubuntu-openstack.starface.local

\# set any password for [adminpassword]
keystone-manage bootstrap --bootstrap-password adminpassword \
--bootstrap-admin-url https://$controller:5000/v3/ \
--bootstrap-internal-url https://$controller:5000/v3/ \
--bootstrap-public-url https://$controller:5000/v3/ \
--bootstrap-region-id RegionOne

nano /etc/apache2/apache2.conf

\# line 70 : add to specify server name
ServerName ubuntu-openstack.starfleet.local

nano /etc/hosts

127.0.0.1 localhost
127.0.1.1 ubuntu-openstack
192.168.200.165 ubuntu-openstack.starfleet.local
\# The following lines are desirable for IPv6 capable hosts
\#::1     ip6-localhost ip6-loopback
\#fe00::0 ip6-localnet
\#ff00::0 ip6-mcastprefix
\#ff02::1 ip6-allnodes
\#ff02::2 ip6-allrouters

mkdir /etc/ssl/ubuntu-openstack

make-ssl-cert generate-default-snakeoil --force-overwrite

make-ssl-cert /usr/share/ssl-cert/ssleay.cnf /etc/ssl/ubuntu-openstack/ubuntu-openstack.starfleet.local.crt

cd /etc/ssl/ubuntu-openstack

\# Extrahiere Zertifikat
awk '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/' ubuntu-openstack.starfleet.local.crt > cert.pem

\# Extrahiere privaten Schlüssel
awk '/-----BEGIN PRIVATE KEY-----/,/-----END PRIVATE KEY-----/' ubuntu-openstack.starfleet.local.crt > key.pem

nano /etc/apache2/sites-available/keystone.conf

\# add settings for SSL/TLS
Listen 5000

<VirtualHost *:5000>
    SSLEngine on
    SSLHonorCipherOrder on
    SSLCertificateFile /etc/ssl/ubuntu-openstack/cert.pem
    SSLCertificateKeyFile /etc/ssl/ubuntu-openstack/key.pem
    SSLCertificateChainFile /etc/ssl/ubuntu-openstack/chain.pem
