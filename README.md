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

  
