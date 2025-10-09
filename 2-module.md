## HQ-SRV
### Нужно создать 2 новых диска
```
apt-get update && apt-get install -y mdadm nfs-server && sleep 5
mdadm --create --verbose /dev/md0 --level=0 --raid-devices=2 /dev/sd[b-c]
mdadm --detail --scan | tee -a /etc/mdadm.conf
apt-get install fdisk -y
echo -e "n\n\n\n\n\nw" | fdisk /dev/md0
mkfs.ext4 /dev/md0p1
mkdir /raid
echo "/dev/md0p1 /raid ext4 defaults 0 0" | tee -a /etc/fstab
mount -a
df -h | grep /raid
mkdir /raid/nfs
chown 99:99 /raid/nfs
chmod 777 /raid/nfs
echo "/raid/nfs 192.168.2.10/24(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -ra
systemctl enable nfs
systemctl restart nfs
sleep 2
apt-get install -y chrony
echo "server 172.16.1.1 iburst prefer" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
timedatectl
```

## HQ-CLI
```
apt-get update && apt-get install -y nfs-clients && sleep 2
mkdir -p /mnt/nfs
echo "192.168.0.10:/raid/nfs /mnt/nfs nfs defaults,auto,_netdev 0 0" >> /etc/fstab
mount -a
df -h | grep /mnt/nfs
showmount -e 192.168.1.10
sleep 2
apt-get install -y chrony
echo "server 172.16.1.1 iburst prefer" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
timedatectl
```
```
sleep 1
useradd remote_user -u 2026 && echo "P@ssw0rd" | passwd --stdin remote_user && gpasswd -a "remote_user" wheel && echo '%wheel ALL=(ALL:ALL) NOPASSWD: ALL' > /etc/sudoers.d/99-wheel-nopasswd
echo -e "Port 2026\nMaxAuthTries 2\nPasswordAuthentication yes\nAllowUsers remote_user\nBanner /etc/openssh/sshd_config" > /etc/openssh/sshd_config
echo -e "Authorized access only!" > /etc/openssh/banner
systemctl restart sshd
systemctl restart network
```

## BR-SRV
```
echo "nameserver 192.168.1.10" > /etc/resolv.conf
echo "192.168.3.10" >> /etc/hosts
apt-get install -y chrony
echo "server 172.16.2.1 iburst prefer" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
timedatectl
sleep 1
```
```
apt-get update && apt-get install ansible sshpass -y
echo -e "[servers]\nHQ-SRV ansible_host=192.168.1.10\nHQ-CLI ansible_host=192.168.2.10\n[servers:vars]\nansible_user=remote_user\nansible_port=2026\n[routers]\nHQ-RTR ansible_host=192.168.1.1\nBR-RTR ansible_host=192.168.3.1\n[routers:vars]\nansible_user=net_admin\nansible_password=P@ssw0rd\nansible_connection=network_cli\nansible_network_os=ios" > /etc/ansible/hosts
echo -e "[defaults]\npython_interpreter=auto_silent" > /etc/ansible/ansible.cfg

ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa -q
sshpass -p 'P@ssw0rd' ssh-copy-id -p 2026 remote_user@192.168.1.10 -f -o StrictHostKeyChecking=no
sshpass -p 'P@ssw0rd' ssh-copy-id -p 2026 remote_user@192.168.2.10 -f -o StrictHostKeyChecking=no
ansible all -m ping
```
```
apt-get update && apt-get install -y docker-compose docker-engine
systemctl enable --now docker
sleep 2
mount -o loop /dev/sr0
docker load < /media/ALTLinux/docker/site_latest.tar
docker load < /media/ALTLinux/docker/mariadb_latest.tar
echo -e "services:\n  db:\n    image: mariadb\n    container_name: db\n    environment:\n      MYSQL_ROOT_PASSWORD: Passw0rd\n      MYSQL_DATABASE: testdb\n      MYSQL_USER: test\n      MYSQL_PASSWORD: Passw0rd\n    volumes:\n      - db_data:/var/lib/mysql\n    restart: always\n\n  testapp:\n    image: site\n    container_name: testapp\n    environment:\n      DB_TYPE: maria\n      DB_HOST: db\n      DB_NAME: testdb\n      DB_USER: test\n      DB_PASS: Passw0rd\n      DB_PORT: 3306\n    ports:\n      - \"8080:8000\"\n    restart: always\n\nvolumes:\n  db_data:" > docker-compose.yaml
sleep 1
docker compose up -d && sleep 5 && docker exec -it db mysql -u root -p'Passw0rd' -e "CREATE DATABASE IF NOT EXISTS testdb; CREATE USER 'test'@'%' IDENTIFIED BY 'Passw0rd'; GRANT ALL PRIVILEGES ON testdb.* TO 'test'@'%'; FLUSH PRIVILEGES;"
```
> [!TIP]
> Проверка что сайт точно работает: 
> curl http://192.168.3.10:8080

## ISP
```
apt-get install chrony –y
echo "server 127.0.0.1 iburst prefer\nhwtimestamp *\nlocal stratum 5\nallow 0/0" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
```
## BR-RTR
```
en
conf t
int tunnel.0
ntp server 172.16.2.1
ntp timezone utc+5
exit
show ntp status
write memory
exit
```
