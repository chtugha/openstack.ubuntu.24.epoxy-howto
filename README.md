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
bind-address          = 0.0.0.0

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

export controller=ubuntu-openstack.starfleet.local

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

a2enmod ssl

cp /etc/ssl/ubuntu-openstack/cert.pem /usr/local/share/ca-certificates/ubuntu-openstack.crt

update-ca-certificates

systemctl restart apache2

nano ~/keystonerc

export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=adminpassword    
export OS_AUTH_URL=https://ubuntu-openstack.starfleet.local:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export PS1='\u@\h \W(keystone)\$ '

chmod 600 ~/keystonerc

source ~/keystonerc

echo "source ~/keystonerc " >> ~/.bashrc

openstack project create --domain default --description "Service Project" service

openstack user create --domain default --project service --password servicepassword glance

openstack role add --project service --user glance admin

openstack service create --name glance --description "OpenStack Image service" image

export controller=ubuntu-openstack.starfleet.local

openstack endpoint create --region RegionOne image public https://$controller:9292

openstack endpoint create --region RegionOne image internal https://$controller:9292

openstack endpoint create --region RegionOne image admin https://$controller:9292

mysql

create database glance;
grant all privileges on glance.* to glance@'localhost' identified by 'servicepassword';
grant all privileges on glance.* to glance@'%' identified by 'servicepassword';
exit

apt -y install glance

mv /etc/glance/glance-api.conf /etc/glance/glance-api.conf.org

nano /etc/glance/glance-api.conf

\# create new
[DEFAULT]
bind_host = 127.0.0.1
\# RabbitMQ connection info
transport_url = rabbit://openstack:password@ubuntu-openstack.starfleet.local:5672
enabled_backends = fs:file

[glance_store]
default_backend = fs

[fs]
filesystem_store_datadir = /var/lib/glance/images/

[database]
\# MariaDB connection info
connection = mysql+pymysql://glance:servicepassword@ubuntu-openstack.starfleet.local:3306/glance

\# keystone auth info
[keystone_authtoken]
www_authenticate_uri = https://ubuntu-openstack.starfleet.local:5000
auth_url = https://ubuntu-openstack.starfleet.local:5000
memcached_servers = ubuntu-openstack.starfleet.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = servicepassword
\# if using self-signed certs on Apache2 Keystone, turn to [true]
insecure = true 

[paste_deploy]
flavor = keystone

[oslo_policy]
enforce_new_defaults = true



chmod 640 /etc/glance/glance-api.conf
chown root:glance /etc/glance/glance-api.conf
su -s /bin/bash glance -c "glance-manage db_sync"
systemctl restart glance-api
systemctl enable glance-api

nano /etc/nginx/nginx.conf

\# add to last line
stream {
    upstream glance-api {
        server 127.0.0.1:9292;
    }
    server {
        listen 192.168.200.165:9292 ssl;
        proxy_pass glance-api;
    }
    ssl_certificate "/etc/ssl/ubuntu-openstack/cert.pem";
    ssl_certificate_key "/etc/ssl/ubuntu-openstack/key.pem";
}


systemctl restart nginx

cd /opt

wget http://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img

modprobe nbd

qemu-nbd --connect=/dev/nbd0 ubuntu-24.04-server-cloudimg-amd64.img

mount /dev/nbd0p1 /mnt

nano /mnt/etc/cloud/cloud.cfg

# line 13 : add
disable_root: false
ssh_pwauth: true

# line 96 : change
lock_passwd: False


umount /mnt

qemu-nbd --disconnect /dev/nbd0p1

openstack image create "Ubuntu2404" --file ubuntu-24.04-server-cloudimg-amd64.img --disk-format qcow2 --container-format bare --public

openstack image list

openstack user create --domain default --project service --password servicepassword nova

openstack role add --project service --user nova admin

openstack user create --domain default --project service --password servicepassword placement

openstack role add --project service --user placement admin

openstack service create --name nova --description "OpenStack Compute service" compute

openstack service create --name placement --description "OpenStack Compute Placement service" placement

export controller=ubuntu-openstack.starfleet.local

openstack endpoint create --region RegionOne compute public https://$controller:8774/v2.1/%\(tenant_id\)s

openstack endpoint create --region RegionOne compute internal https://$controller:8774/v2.1/%\(tenant_id\)s

openstack endpoint create --region RegionOne compute admin https://$controller:8774/v2.1/%\(tenant_id\)s

openstack endpoint create --region RegionOne placement public https://$controller:8778

openstack endpoint create --region RegionOne placement internal https://$controller:8778

openstack endpoint create --region RegionOne placement admin https://$controller:8778

mysql

create database nova;
grant all privileges on nova.* to nova@'localhost' identified by 'password';
grant all privileges on nova.* to nova@'%' identified by 'password';
create database nova_api;
grant all privileges on nova_api.* to nova@'localhost' identified by 'password';
grant all privileges on nova_api.* to nova@'%' identified by 'password';
create database placement;
grant all privileges on placement.* to placement@'localhost' identified by 'password';
grant all privileges on placement.* to placement@'%' identified by 'password';
create database nova_cell0;
grant all privileges on nova_cell0.* to nova@'localhost' identified by 'password';
grant all privileges on nova_cell0.* to nova@'%' identified by 'password';
exit

apt -y install nova-api nova-conductor nova-scheduler nova-novncproxy placement-api python3-novaclient

mv /etc/nova/nova.conf /etc/nova/nova.conf.org

nano /etc/nova/nova.conf

# create new
[DEFAULT]
osapi_compute_listen = 127.0.0.1
osapi_compute_listen_port = 8774
metadata_listen = 127.0.0.1
metadata_listen_port = 8775
state_path = /var/lib/nova
enabled_apis = osapi_compute,metadata
log_dir = /var/log/nova
# RabbitMQ connection info
transport_url = rabbit://openstack:password@ubuntu-openstack.starfleet.local:5672

[api]
auth_strategy = keystone

[vnc]
enabled = True
novncproxy_host = 127.0.0.1
novncproxy_port = 6080
novncproxy_base_url = https://ubuntu-openstack.starfleet.local:6080/vnc_auto.html

# Glance connection info
[glance]
api_servers = https://ubuntu-openstack.starfleet.local:9292

[oslo_concurrency]
lock_path = $state_path/tmp

# MariaDB connection info
[api_database]
connection = mysql+pymysql://nova:password@ubuntu-openstack.starfleet.local:3306/nova_api

[database]
connection = mysql+pymysql://nova:password@ubuntu-openstack.starfleet.local:3306/nova

# Keystone auth info
[keystone_authtoken]
www_authenticate_uri = https://ubuntu-openstack.starfleet.local:5000
auth_url = https://ubuntu-openstack.starfleet.local:5000
memcached_servers = ubuntu-openstack.starfleet.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = servicepassword
# if using self-signed certs on Apache2 Keystone, turn to [true]
insecure = true

[placement]
auth_url = https://ubuntu-openstack.starfleet.local:5000
os_region_name = RegionOne
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = servicepassword
# if using self-signed certs on Apache2 Keystone, turn to [true]
insecure = true

[wsgi]
api_paste_config = /etc/nova/api-paste.ini

[oslo_policy]
enforce_new_defaults = true
