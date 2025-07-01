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


chmod 640 /etc/nova/nova.conf

chgrp nova /etc/nova/nova.conf

mv /etc/placement/placement.conf /etc/placement/placement.conf.org

nano /etc/placement/placement.conf

# create new
[DEFAULT]
debug = false

[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = https://ubuntu-openstack.starfleet.local:5000
auth_url = https://ubuntu-openstack.starfleet.local:5000
memcached_servers = ubuntu-openstack.starfleet.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = servicepassword
# if using self-signed certs on Apache2 Keystone, turn to [true]
insecure = true

[placement_database]
connection = mysql+pymysql://placement:password@ubuntu-openstack.starfleet.local:3306/placement



nano /etc/apache2/sites-enabled/placement-api.conf

# line 1 : change
Listen 127.0.0.1:8778


chmod 640 /etc/placement/placement.conf

chgrp placement /etc/placement/placement.conf

nano /etc/nginx/nginx.conf

# change the [stream] section
stream {
    upstream glance-api {
        server 127.0.0.1:9292;
    }
    server {
        listen 192.168.200.165:9292 ssl;
        proxy_pass glance-api;
    }
    upstream nova-api {
        server 127.0.0.1:8774;
    }
    server {
        listen 192.168.200.165:8774 ssl;
        proxy_pass nova-api;
    }
    upstream nova-metadata-api {
        server 127.0.0.1:8775;
    }
    server {
        listen 192.168.200.165:8775 ssl;
        proxy_pass nova-metadata-api;
    }
    upstream placement-api {
        server 127.0.0.1:8778;
    }
    server {
        listen 192.168.200.165:8778 ssl;
        proxy_pass placement-api;
    }
    upstream novncproxy {
        server 127.0.0.1:6080;
    }
    server {
        listen 192.168.200.165:6080 ssl;
        proxy_pass novncproxy;
    }
    ssl_certificate "/etc/ssl/ubuntu-openstack/cert.pem";
    ssl_certificate_key "/etc/ssl/ubuntu-openstack/key.pem";
}


su -s /bin/bash placement -c "placement-manage db sync"

su -s /bin/bash nova -c "nova-manage api_db sync"

su -s /bin/bash nova -c "nova-manage cell_v2 map_cell0"

su -s /bin/bash nova -c "nova-manage db sync"

su -s /bin/bash nova -c "nova-manage cell_v2 create_cell --name cell1"

systemctl restart nova-api nova-conductor nova-scheduler nova-novncproxy

systemctl enable nova-api nova-conductor nova-scheduler nova-novncproxy

systemctl restart apache2 nginx

openstack compute service list

apt -y install nova-compute nova-compute-kvm

nano /etc/nova/nova.conf

# add into the [vnc] section
# IP address compute instances listen
[vnc]
enabled = True
server_listen = 192.168.200.165
server_proxyclient_address = 192.168.200.165
novncproxy_host = 127.0.0.1
novncproxy_port = 6080
novncproxy_base_url = https://ubuntu-openstack.starfleet.local:6080/vnc_auto.html


systemctl restart nova-compute

su -s /bin/bash nova -c "nova-manage cell_v2 discover_hosts"

openstack compute service list

openstack user create --domain default --project service --password servicepassword neutron

openstack role add --project service --user neutron admin

openstack service create --name neutron --description "OpenStack Networking service" network

export controller=ubuntu-openstack.starfleet.local

openstack endpoint create --region RegionOne network public https://$controller:9696

openstack endpoint create --region RegionOne network internal https://$controller:9696

openstack endpoint create --region RegionOne network admin https://$controller:9696

mysql

create database neutron_ml2;
grant all privileges on neutron_ml2.* to neutron@'localhost' identified by 'password';
grant all privileges on neutron_ml2.* to neutron@'%' identified by 'password';
exit


apt -y install neutron-server neutron-plugin-ml2 neutron-ovn-metadata-agent python3-neutronclient ovn-central ovn-host

mv /etc/neutron/neutron.conf /etc/neutron/neutron.conf.org

nano /etc/neutron/neutron.conf

# create new
[DEFAULT]
bind_host = 127.0.0.1
bind_port = 9696
core_plugin = ml2
service_plugins = ovn-router
auth_strategy = keystone
state_path = /var/lib/neutron
allow_overlapping_ips = True
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
# RabbitMQ connection info
transport_url = rabbit://openstack:password@ubuntu-openstack.starfleet.local:5672

# Keystone auth info
[keystone_authtoken]
www_authenticate_uri = https://ubuntu-openstack.starfleet.local:5000
auth_url = https://ubuntu-openstack.starfleet.local:5000
memcached_servers = ubuntu-openstack.starfleet.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = servicepassword
# if using self-signed certs on Apache2 Keystone, turn to [true]
insecure = true

[database]
connection = mysql+pymysql://neutron:password@ubuntu-openstack.starfleet.local:3306/neutron_ml2

[nova]
auth_url = https://ubuntu-openstack.starfleet.local:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = servicepassword
# if using self-signed certs on Apache2 Keystone, turn to [true]
insecure = true

[oslo_concurrency]
lock_path = $state_path/tmp

[oslo_policy]
enforce_new_defaults = true


chmod 640 /etc/neutron/neutron.conf

chgrp neutron /etc/neutron/neutron.conf

mv /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.org

nano /etc/neutron/plugins/ml2/ml2_conf.ini

# create new
[DEFAULT]
debug = false

[ml2]
type_drivers = flat,geneve
tenant_network_types = geneve
mechanism_drivers = ovn
extension_drivers = port_security
overlay_ip_version = 4

[ml2_type_geneve]
vni_ranges = 1:65536
max_header_size = 38

[ml2_type_flat]
flat_networks = *

[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

[ovn]
ovn_nb_connection = tcp:192.168.200.165:6641
ovn_sb_connection = tcp:192.168.200.165:6642
ovn_l3_scheduler = leastloaded
ovn_metadata_enabled = True


chmod 640 /etc/neutron/plugins/ml2/ml2_conf.ini

chgrp neutron /etc/neutron/plugins/ml2/ml2_conf.ini

nano /etc/neutron/neutron_ovn_metadata_agent.ini

[DEFAULT]
# line 2 : add to specify Nova API host
nova_metadata_host = ubuntu-openstack.starfleet.local
nova_metadata_protocol = https
# specify any secret key you like
metadata_proxy_shared_secret = metadata_secret

# line 272 : change
[ovs]
ovsdb_connection = tcp:127.0.0.1:6640

# add to last line
[agent]
root_helper = sudo neutron-rootwrap /etc/neutron/rootwrap.conf

[ovn]
ovn_sb_connection = tcp:192.168.200.165:6642


nano /etc/default/openvswitch-switch

# line 8 : uncomment and add like follows
OVS_CTL_OPTS="--ovsdb-server-options='--remote=ptcp:6640:127.0.0.1'"


nano /etc/nova/nova.conf

# add follows into the [DEFAULT] section
vif_plugging_is_fatal = True
vif_plugging_timeout = 300

# add follows to last line : Neutron auth info
# the value of [metadata_proxy_shared_secret] is the same with the one in [metadata_agent.ini]
[neutron]
auth_url = https://ubuntu-openstack.starfleet.local:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = servicepassword
service_metadata_proxy = True
metadata_proxy_shared_secret = metadata_secret
insecure = true


nano /etc/nginx/nginx.conf

# add into the [stream] section
upstream neutron-api {
        server 127.0.0.1:9696;
    }
    server {
        listen 192.168.200.165:9696 ssl;
        proxy_pass neutron-api;
    }



ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini

su -s /bin/bash neutron -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugin.ini upgrade head"

systemctl restart ovn-central ovn-northd ovn-controller ovn-host

ovn-nbctl set-connection ptcp:6641:192.168.200.165 -- set connection . inactivity_probe=60000

ovn-sbctl set-connection ptcp:6642:192.168.200.165 -- set connection . inactivity_probe=60000

ovs-vsctl set open . external-ids:ovn-remote=tcp:192.168.200.165:6642

ovs-vsctl set open . external-ids:ovn-encap-type=geneve

ovs-vsctl set open . external-ids:ovn-encap-ip=192.168.200.165

ovs-vsctl set open . external-ids:ovn-cms-options=enable-chassis-as-gw

systemctl restart neutron-server neutron-ovn-metadata-agent nova-api nova-compute nginx

openstack network agent list

nano /etc/systemd/network/dummy0.netdev

[NetDev]
Name=dummy0
Kind=dummy


systemctl restart systemd-networkd

cp /etc/netplan/01-netcfg.yaml /etc/netplan/01-netcfg.yaml.org

nano /etc/netplan/01-netcfg.yaml

network:
  version: 2
  renderer: networkd

  ethernets:
    ens192: {}     # für br-mgmt (Management)
    ens224: {}     # für br-ex (VLAN-Trunk ohne IP)
    dummy0: {}
    
  bonds:
    bond0:
      interfaces:
        - ens224  # add more if you have more uplink LACP Ports
        - dummy0
      parameters:
        mode: balance-slb
      # by default the vlan is already untagged, no extra option needed here.

  bridges:
    br-mgmt:
      interfaces: [ens192]
      addresses: [192.168.200.165/24]
      routes:
        - to: default
          via: 192.168.200.1
          metric: 100
      mtu: 1500
      nameservers:
        addresses: [192.168.200.1]
        search: [starfleet.local]
      parameters:
        stp: false
        forward-delay: 0
      dhcp6: false
      dhcp4: false
    br-ex:
      interfaces:
        - bond0
  vlans:
    vlan2:
      id: 2
      link: br-ex
      mtu: 1500
      addresses: []
    vlan16:
      id: 16
      link: br-ex
      mtu: 1500
      addresses: []
    vlan17:
      id: 17
      link: br-ex
      mtu: 1500
      addresses: []
    vlan18:
      id: 18
      link: br-ex
      mtu: 1500
      addresses: []
    vlan19:
      id: 19
      link: br-ex
      mtu: 1500
      addresses: [192.168.19.123/24]
      routes:
        - to: default
          via: 192.168.19.1
          metric: 101
      openvswitch: {}
      nameservers:
        addresses: [192.168.200.1]
        search: [outside]
      dhcp4: false
      dhcp6: false
    vlan20:
      id: 20
      link: br-ex
      mtu: 9000
      addresses: []
    vlan22:
      id: 22
      link: br-ex
      mtu: 9000
      addresses: []


nano /etc/neutron/plugins/ml2/ml2_conf.ini

# create new
[DEFAULT]
debug = false
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:password@ubuntu-openstack.starfleet.local

[ml2]
type_drivers = local,flat,geneve,vlan
tenant_network_types = geneve,vlan  
mechanism_drivers = ovn
extension_drivers = port_security
overlay_ip_version = 4

#[ovs]
#bridge_mappings = physnet1:br-ex    

[ml2_type_vlan]
network_vlan_ranges = physnet1:2:22               

#[ml2_type_vxlan]
#vni_ranges = 1001:2000
# Optional: local_ip is defined in openvswitch_agent.ini

[ml2_type_geneve]
vni_ranges = 1:65536
max_header_size = 38

[ml2_type_flat]
flat_networks = *

[securitygroup]
enable_security_group = True
#firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
firewall_driver = ovn

[ovn]
ovn_nb_connection = tcp:192.168.200.165:6641
ovn_sb_connection = tcp:192.168.200.165:6642
ovn_l3_scheduler = leastloaded
ovn_metadata_enabled = True



mv /etc/neutron/plugins/ml2/openvswitch_agent.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini.org

systemctl restart neutron-server ovn-controller

ovs-vsctl set open . external_ids:ovn-bridge-mappings=physnet1:br-ex

ovs-vsctl set port bond0 trunks=2-22

openstack network create --provider-network-type vlan --provider-physical-network physnet1 --provider-segment 19 public-vlan19

openstack subnet create --network public-vlan19 --subnet-range 192.168.19.0/24 --gateway 192.168.19.1 public-subnet19

openstack user create --domain default --project service --password servicepassword cinder

openstack role add --project service --user cinder admin

openstack role add --project service --user cinder service

openstack role add --project service --user nova service

openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3

export controller=ubuntu-openstack.starfleet.local

openstack endpoint create --region RegionOne volumev3 public https://$controller:8776/v3/%\(tenant_id\)s

openstack endpoint create --region RegionOne volumev3 internal https://$controller:8776/v3/%\(tenant_id\)s

openstack endpoint create --region RegionOne volumev3 admin https://$controller:8776/v3/%\(tenant_id\)s

mysql

create database cinder;
grant all privileges on cinder.* to cinder@'localhost' identified by 'password';
grant all privileges on cinder.* to cinder@'%' identified by 'password';
exit

apt -y install cinder-api cinder-scheduler python3-cinderclient

mv /etc/cinder/cinder.conf /etc/cinder/cinder.conf.org

nano /etc/cinder/cinder.conf

# create new
[DEFAULT]
rootwrap_config = /etc/cinder/rootwrap.conf
api_paste_confg = /etc/cinder/api-paste.ini
state_path = /var/lib/cinder
auth_strategy = keystone
# RabbitMQ connection info
transport_url = rabbit://openstack:password@ubuntu-openstack.starfleet.local:5672
enable_v3_api = True

# MariaDB connection info
[database]
connection = mysql+pymysql://cinder:password@ubuntu-openstack.starfleet.local:3306/cinder

# Keystone auth info
[service_user]
send_service_user_token = true
auth_url = https://ubuntu-openstack.starfleet.local:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = cinder
password = servicepassword
# if using self-signed certs on Apache2 Keystone, turn to [true]
insecure = true

# Keystone auth info
[keystone_authtoken]
www_authenticate_uri = https://ubuntu-openstack.starfleet.local:5000
auth_url = https://ubuntu-openstack.starfleet.local:5000
memcached_servers = ubuntu-openstack.starfleet.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = cinder
password = servicepassword
# if using self-signed certs on Apache2 Keystone, turn to [true]
insecure = false

[oslo_concurrency]
lock_path = $state_path/tmp

[oslo_policy]
enforce_new_defaults = true

[enabled_backends]
#add name of iscsi connection after lvm: lvm,synology
enabled_backends = synology

#[lvm]
#target_helper = lioadm
#target_protocol = iscsi
#target_ip_address = $my_ip
# volume group name created on [1]
#volume_group = vg_volume01
#volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
#volumes_dir = $state_path/volumes

[synology]
volume_driver=cinder.volume.drivers.synology.synology_iscsi.SynoISCSIDriver
volume_backend_name=synology   
target_protocol=iscsi
target_ip_address=192.168.10.11
synology_admin_port=5000
synology_username=Administrator
synology_password=*****
synology_pool_name = volume1





chmod 640 /etc/cinder/cinder.conf

chgrp cinder /etc/cinder/cinder.conf

nano /etc/apache2/conf-available/cinder-wsgi.conf

# line 1 : change
Listen 127.0.0.1:8776


nano /etc/nginx/nginx.conf

# add under stream:
    upstream cinder-api {
        server 127.0.0.1:8776;
    }
    server {
        listen 192.168.200.165:8776 ssl;
        proxy_pass cinder-api;
    }


su -s /bin/bash cinder -c "cinder-manage db sync"

systemctl restart cinder-scheduler apache2 nginx

systemctl enable cinder-scheduler

echo "export OS_VOLUME_API_VERSION=3" >> ~/keystonerc

source ~/keystonerc

apt -y install cinder-volume python3-mysqldb targetcli-fb python3-rtslib-fb

systemctl enable cinder-volume

nano /etc/nova/nova.conf

# add follows into the [keystone_authtoken] section
service_token_roles = service
service_token_roles_required = true

# add to last line
[service_user]
send_service_user_token = true
auth_url = https://ubuntu-openstack.starfleet.local:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = servicepassword
insecure = false

[cinder]
os_region_name = RegionOne


apt -y install openstack-dashboard

nano /etc/openstack-dashboard/local_settings.py

# line 99 : change Memcache server
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '192.168.200.165:11211',
    },
}

# line 113 : add
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
# line 126 : set Openstack Host
# line 127 : comment out and add a line to specify URL of Keystone Host
OPENSTACK_HOST = "ubuntu-openstack.starfleet.local"
#OPENSTACK_KEYSTONE_URL = "http://%s/identity/v3" % OPENSTACK_HOST"
OPENSTACK_KEYSTONE_URL = "https://ubuntu-openstack.starfleet.local:5000/v3"
# line 131 : set your timezone
TIME_ZONE = "Europe/Berlin"
# add to the end
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'

nano /etc/apache2/sites-available/default-ssl.conf
# line 32,33 : change to your certificate
SSLCertificateFile      /etc/ssl/ubuntu-openstack/cert.pem
SSLCertificateKeyFile   /etc/ssl/ubuntu-openstack/key.pem

# line 42 : uncomment and specify your chain file
SSLCertificateChainFile /etc/ssl/ubuntu-openstack/cert.pem

a2enmod ssl

a2ensite default-ssl

systemctl restart apache2 nova-api
