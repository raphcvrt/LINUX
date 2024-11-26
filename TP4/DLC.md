# DLC

# Etape 1
1. je recherche des utilisateurs récemment ajoutés :

```sh
[root@localhost secure_data]# sudo cat /etc/passwd
attacker:x:1000:1000::/home/attacker:/bin/bash
# Entre autres
```

2. je cherche les fichiers récemment modifiés dans des répertoires critiques :

```sh
[root@localhost ~]# sudo find /etc /usr/local/bin /var -type f -mtime -7 | grep secure
/etc/lvm/backup/vg_secure
/etc/lvm/archive/vg_secure_00001-428914066.vg
/etc/lvm/archive/vg_secure_00003-423511421.vg
/etc/lvm/archive/vg_secure_00000-1860456324.vg
/etc/lvm/archive/vg_secure_00002-656608851.vg
/var/log/secure
```

3. Je liste les services suspects activés :

```sh
[root@localhost ~]# sudo systemctl list-unit-files --state=enabled
UNIT FILE                          STATE   PRESET  
auditd.service                     enabled enabled 
chronyd.service                    enabled enabled 
crond.service                      enabled enabled 
dbus-broker.service                enabled enabled 
firewalld.service                  enabled enabled 
getty@.service                     enabled enabled 
irqbalance.service                 enabled enabled 
kdump.service                      enabled enabled 
lvm2-monitor.service               enabled enabled 
microcode.service                  enabled enabled 
NetworkManager-dispatcher.service  enabled enabled 
NetworkManager-wait-online.service enabled disabled
NetworkManager.service             enabled enabled 
nis-domainname.service             enabled enabled 
rsyslog.service                    enabled enabled 
selinux-autorelabel-mark.service   enabled enabled 
sshd.service                       enabled enabled 
sssd.service                       enabled enabled 
systemd-boot-update.service        enabled enabled 
systemd-network-generator.service  enabled enabled 
dbus.socket                        enabled enabled 
dm-event.socket                    enabled enabled 
lvm2-lvmpolld.socket               enabled enabled 
sssd-kcm.socket                    enabled enabled 
reboot.target                      enabled enabled 
remote-fs.target                   enabled enabled 
dnf-makecache.timer                enabled enabled 
logrotate.timer                    enabled enabled 
```

4. Je supprime une tâche cron suspecte :

```sh
[root@localhost ~]# ls /var/spool/cron/
attacker
[root@localhost ~]# sudo crontab -u attacker -r
```

# Etape 2

1. je cree un snapshot du lv :

```sh
[root@localhost ~]# sudo lvcreate --snapshot --name secure_data_snapshot --size 500M /dev/vg_secure/secure_data
  Rounding up size to full physical extent 500.00 MiB
  Logical volume "secure_data_snapshot" created.
```

2. Je le teste :

```sh
[root@localhost ~]# sudo mount /dev/vg_secure/secure_data_snapshot /mnt/secure_data_snapshot

[root@localhost ~]# sudo ls /mnt/secure_data_snapshot
lost+found  sensitive1.txt  sensitive2.txt

```

3. je simule une restauration :

```sh
[root@localhost ~]# sudo rm /mnt/secure_data/sensitive1.txt
[root@localhost secure_data]# ls
sensitive2.txt  testfile1.txt  testfile2.txt
[root@localhost ~]# sudo cp /mnt/secure_data_snapshot/sensitive2.txt /mnt/secure_data/
[root@localhost secure_data]# ls
sensitive1.txt  sensitive2.txt  testfile1.txt  testfile2.txt
```

# Etape 3

1. je bloque les bruteforce :

```sh
[root@localhost secure_data]# sudo firewall-cmd --permanent --add-rich-rule="rule family='ipv4' service name='ssh' limit value='2/m' accept"
success
[root@localhost secure_data]# sudo firewall-cmd --reload
success
[root@localhost secure_data]# sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3 enp0s8
  sources: 
  services: http https ssh
  ports: 2222/tcp
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
        rule family="ipv4" source address="192.168.1.0/24" service name="ssh" accept
        rule family="ipv4" service name="ssh" accept limit value="2/m"

```

2. Je restrein l’accès SSH à une range spécifique d'ip :

```sh
deja fait

[root@localhost secure_data]# sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3 enp0s8
  sources: 
  services: http https ssh
  ports: 2222/tcp
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
        rule family="ipv4" source address="192.168.1.0/24" service name="ssh" accept
        rule family="ipv4" source address="192.168.0.0/16" service name="ssh" accept
        rule family="ipv4" service name="ssh" accept limit value="2/m"

```

3. Je crée une zone sécurisée pour un service web :

```sh
[root@localhost secure_data]# sudo firewall-cmd --permanent --new-zone=web_zone
success

[root@localhost secure_data]# sudo firewall-cmd --permanent --zone=web_zone --add-service=http
success

[root@localhost secure_data]# sudo firewall-cmd --permanent --zone=web_zone --add-service=https
success

[root@localhost secure_data]# sudo firewall-cmd --permanent --zone=web_zone --set-target=DROP
success


[root@localhost secure_data]# sudo firewall-cmd --permanent --zone=web_zone --change-interface=enp0s8
success

[root@localhost secure_data]# sudo firewall-cmd --reload
success

[root@localhost secure_data]# sudo firewall-cmd --zone=web_zone --list-all
web_zone
  target: DROP
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: http https
  ports: 
  protocols: 
  forward: no
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

```

# Etape 4

1. je crée un script monitor.sh et j'y inclus une alerte par e-mail :

3. je le rajoute au crontab :

```sh
[root@localhost ~]# sudo crontab -e
```
j'ajoute dans le crontab:
```sh
*/5 * * * * /usr/local/bin/monitor.sh >> /var/log/monitor_cron.log 2>&1
```

# Etape 5

1. Je configure AIDE :

```sh
[root@localhost ~]# aide --version

[root@localhost ~]# sudo aide --init

[root@localhost ~]# sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db

[root@localhost ~]# sudo nano /etc/aide.conf
```
je rajoute ca a la suite

```sh
/etc           p+i+n+u+g+s+b+m+c+md5+sha512
/bin           p+i+n+u+g+s+b+m+c+md5+sha512
/sbin          p+i+n+u+g+s+b+m+c+md5+sha512
/usr/bin       p+i+n+u+g+s+b+m+c+md5+sha512
/usr/sbin      p+i+n+u+g+s+b+m+c+md5+sha512
```

2. Je teste :

```sh
[root@localhost ~]# sudo touch /etc/testfile

[root@localhost ~]# sudo aide --check

la modification est signalée dans la sortie.
```
