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

# line 95 : confirm default charset
# if use 4 bytes UTF-8, specify [utf8mb4]
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

