# rsyslog
## HQ-SRV
```powershell
apt-get update && apt-get install rsyslog logrotate -y
sudo sed -i '/^#module(load="im[ut][dc]p")/s/^#//; /^#input(type="im[ut][dc]p"/s/^#//' /etc/rsyslog.d/00_common.conf
cat << 'EOF' >> /etc/rsyslog.d/00_common.conf
$template RemoteLogs, "/opt/%HOSTNAME%/%HOSTNAME%.log"
if ($fromhost-ip != '127.0.0.1') then ?RemoteLogs
& stop
EOF
cat << 'EOF' > /etc/logrotate.d/rsyslog
/opt/*/*.log {
    weekly
    size 10M
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
    sharedscripts
    postrotate
        /usr/bin/systemctl reload rsyslog > /dev/null 2>&1 || true
    endscript
}
EOF
sudo bash -c '(crontab -l 2>/dev/null; echo "0 0 * * 0 /usr/sbin/logrotate /etc/logrotate.d/rsyslog") | crontab -'
systemctl enable --now rsyslog
systemctl start --now rsyslog
systemctl status --now rsyslog
echo "172.16.0.2 br-rtr" >> /etc/hosts
systemctl restart network
systemctl restart --now rsyslog
```

## HQ-RTR
```assembly
en
conf t
rsyslog host 192.168.1.10 mode tcp port 514
write
```

## BR-RTR
```assembly
en
conf t
rsyslog host 192.168.1.10 mode tcp port 514
write
```

## BR-SRV
```powershell
apt-get update && apt-get install rsyslog logrotate rsyslog-journal -y
sudo sed -i '/^#module(load="im\(journal\|uxsock\|klog\|mark\)")/s/^#//' /etc/rsyslog.d/00_common.conf
sudo sed -i '/global(workDirectory="\/var\/spool\/rsyslog")/a *.warning @@192.168.1.10:514' /etc/rsyslog.d/00_common.conf
systemctl enable --now rsyslog
systemctl start --now rsyslog
systemctl status --now rsyslog
logger -p user.warn "Test Warning Message"
```

# Проверка
## HQ-SRV
```assembly
ls /opt/
logrotate -d /etc/logrotate.d/rsyslog 
```

> [!WARNING]
> Если не появляется папка с логами для br-rtr, повторять следующее пока не заработает:

## BR-RTR
```powershell
en
telnet 192.168.1.10 port 514
ctrl + ]
send ayt
br-rtr
ctrl + ]
send ayt
br-rtr
```

### Если в файл Hosts не ввести **192.168.0.2 br-rtr** то, логи будут выдавать 192.168.0.2.
