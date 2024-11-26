# Etape 1

1. Je commence par extraire un user suspect parmis tous puis je regarde sa cron table.
```sh
[root@localhost paph]# cat /etc/passwd
attacker:x:1000:1000::/home/attacker:/bin/bash
# Entre autres
[root@localhost paph]# sudo crontab -u attacker -l  
*/10 * * * * /tmp/.hidden_script
```
2. j'identifie et supprime les fichiers suspects
```sh
[root@localhost paph]# cd /tmp/
[root@localhost tmp]# ls -a
.          .X11-unix   .hidden_file    systemd-private-9dcbb28856ef47aa94cfcdfdaff6b0b8-chronyd.service-3ttVX4      systemd-private-9dcbb28856ef47aa94cfcdfdaff6b0b8-kdump.service-zEKMpF
..         .XIM-unix   .hidden_script  systemd-private-9dcbb28856ef47aa94cfcdfdaff6b0b8-dbus-broker.service-frNsYX  systemd-private-9dcbb28856ef47aa94cfcdfdaff6b0b8-systemd-logind.service-3F4I45
.ICE-unix  .font-unix  malicious.sh    systemd-private-9dcbb28856ef47aa94cfcdfdaff6b0b8-irqbalance.service-29xvA8
[root@localhost tmp]# rm .hidden_file .hidden_script malicious.sh
```
3. je regarde les connexions actives avec ss
```sh
[root@localhost tmp]# sudo ss -tunap
Netid            State             Recv-Q            Send-Q                               Local Address:Port                         Peer Address:Port            Process                                                          
udp              ESTAB             0                 0                            192.168.56.102%enp0s8:68                         192.168.56.100:67               users:(("NetworkManager",pid=833,fd=32))                        
udp              ESTAB             0                 0                                 10.0.2.15%enp0s3:68                               10.0.2.2:67               users:(("NetworkManager",pid=833,fd=26))                        
udp              UNCONN            0                 0                                        127.0.0.1:323                               0.0.0.0:*                users:(("chronyd",pid=828,fd=5))                                
udp              UNCONN            0                 0                                            [::1]:323                                  [::]:*                users:(("chronyd",pid=828,fd=6))                                
tcp              LISTEN            0                 128                                        0.0.0.0:22                                0.0.0.0:*                users:(("sshd",pid=853,fd=3))                                   
tcp              ESTAB             0                 0                                   192.168.56.102:22                         192.168.56.100:50846            users:(("sshd",pid=1853,fd=4),("sshd",pid=1849,fd=4))           
tcp              LISTEN            0                 128                                           [::]:22                                   [::]:*                users:(("sshd",pid=853,fd=4))
```
# Etape 2

1. je commence par afficher la liste des logical volumes puis je crée un snapshot pour secure_data
```sh
[root@localhost tmp]# sudo lvdisplay
  --- Logical volume ---
  LV Path                /dev/vg_secure/secure_data
  LV Name                secure_data
  VG Name                vg_secure
  LV UUID                gMLkSZ-8Yhz-m9Hd-1jiQ-4C0G-PRVy-FYqFeJ
  LV Write Access        read/write
  LV Creation host, time vbox, 2024-11-24 18:24:53 +0100
  LV Status              available
  # open                 1
  LV Size                500.00 MiB
  Current LE             125
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:2
   
  --- Logical volume ---
  LV Path                /dev/rl_vbox/swap
  LV Name                swap
  VG Name                rl_vbox
  LV UUID                z3wCgb-7tII-XPA1-fcb9-7arD-fqGE-V1HIqV
  LV Write Access        read/write
  LV Creation host, time vbox, 2024-11-24 17:42:22 +0100
  LV Status              available
  # open                 2
  LV Size                512.00 MiB
  Current LE             128
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:1
   
  --- Logical volume ---
  LV Path                /dev/rl_vbox/root
  LV Name                root
  VG Name                rl_vbox
  LV UUID                WGJSf5-HJZJ-nPZc-bTbS-fP0E-8QgG-jTh5wo
  LV Write Access        read/write
  LV Creation host, time vbox, 2024-11-24 17:42:22 +0100
  LV Status              available
  # open                 1
  LV Size                <3.50 GiB
  Current LE             895
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0
[root@localhost tmp]# sudo lvcreate --size 1G --snapshot --name snap_secure_data /dev/vg_secure/secure_data
  Reducing COW size 1.00 GiB down to maximum usable size 504.00 MiB.
  Logical volume "snap_secure_data" created.
```
2. je supprime sensitive1.txt et sensitive2.txt pour voir
```sh
[root@localhost tmp]# cd /mnt/secure_data/
[root@localhost secure_data]# ls
lost+found  sensitive1.txt  sensitive2.txt
[root@localhost secure_data]# rm sensitive1.txt sensitive2.txt -y
[root@localhost secure_data]# ls
lost+found
```
je monte le snapshot
```sh
[root@localhost secure_data]# sudo mkdir /mnt/snapshot
[root@localhost secure_data]# sudo mount /dev/vg_secure/snap_secure_data /mnt/snapshot/
[root@localhost secure_data]# cd /mnt/snapshot
[root@localhost snapshot]# ls
lost+found  sensitive1.txt  sensitive2.txt
[root@localhost snapshot]# sudo cp /mnt/snapshot/sensitive1.txt /mnt/snapshot/sensitive2.txt /mnt/secure_data/
```
3. (Pour agrandir le lv et le vg j'ai galéré toute la soirée sur des forums j'ai pas réussi meme ChatGPT il a pas resolu après jsuis ptetre un golmon)

# Etape 3
1. le script ressemble a ça et il a été entièrement fait par ChatGPT car je ne connais pas bash, il ne conserve aussi que les 7 dernières sauvegardes
```bash
#!/bin/bash

# Variables
BACKUP_DIR="/backup"
SOURCE_DIR="/mnt/secure_data"
DATE=$(date +%Y%m%d)
ARCHIVE_NAME="secure_data_${DATE}.tar.gz"

# Vérifier si le répertoire de sauvegarde existe, sinon le créer
if [ ! -d "$BACKUP_DIR" ]; then
    echo "Le répertoire $BACKUP_DIR n'existe pas. Création en cours..."
    mkdir -p "$BACKUP_DIR"
fi

# Créer l'archive en excluant les fichiers temporaires, logs et fichiers cachés
echo "Création de l'archive dans $BACKUP_DIR/$ARCHIVE_NAME..."
tar --exclude='*.tmp' --exclude='*.log' --exclude='.*' -czf "$BACKUP_DIR/$ARCHIVE_NAME" "$SOURCE_DIR"
if [ $? -ne 0 ]; then
    echo "Erreur lors de la création de l'archive."
    exit 1
fi
echo "Archive créée avec succès."

# Rotation des sauvegardes : conserver uniquement les 7 dernières
echo "Rotation des sauvegardes : conservation des 7 dernières archives..."
cd "$BACKUP_DIR" || exit 1
ls -tp | grep -v '/$' | tail -n +8 | xargs -d '\n' -r rm --
echo "Rotation terminée."
```
3. je le teste et le verifie 
```bash
[root@localhost home]# ./secure_backup.sh 
Création de l'archive dans /backup/secure_data_20241125.tar.gz...
tar: Removing leading `/' from member names
Archive créée avec succès.
Rotation des sauvegardes : conservation des 7 dernières archives...
Rotation terminée.
[root@localhost home]# ls /backup
secure_data_20241125.tar.gz
```
4. ajout de cette ligne dans le crontab pour que le script s'execute tout les jours de tout les mois a chaque jour de la semaine, à 
3h et 0 minutes
```sh
0 3 * * * /home/secure_backup.sh
```
# Etape 4
1. je rajoute une regle auditctl 
```bash
[root@localhost audit]# auditctl -w /etc -k accès_etc
Old style watch rules are slower
[root@localhost audit]# auditctl -l
-w /etc -p rwxa -k accès_etc
```
2. je crée volontairement un evennement dans le /etc puis je le cherche dans les logs
```sh
[root@localhost audit]# sudo echo "paph sudios" > /etc/test1.txt
[root@localhost audit]# cat /var/log/audit/audit.log | sudo ausearch -k accès_etc > /var/log/audit_etc.log
[root@localhost audit]# cat /var/log/audit_etc.log
----
time->Mon Nov 25 20:28:33 2024
type=PROCTITLE msg=audit(1732562913.983:1314): proctitle=617564697463746C002D77002F657463002D6B00616363C3A8735F657463
type=SOCKADDR msg=audit(1732562913.983:1314): saddr=100000000000000000000000
type=SYSCALL msg=audit(1732562913.983:1314): arch=c000003e syscall=44 success=yes exit=1072 a0=4 a1=7ffd8eeb6320 a2=430 a3=0 items=0 ppid=2310 pid=5739 auid=1001 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=13 comm="auditctl" exe="/usr/sbin/auditctl" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key=(null)
type=CONFIG_CHANGE msg=audit(1732562913.983:1314): auid=1001 ses=13 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 op=add_rule key=616363C3A8735F657463 list=4 res=1

```
# Etape 5
1. je verifie et configure le pare-feu pour les ports ssh htt et https (respectivement 22, 80 et 443)
```sh
[root@localhost audit]# sudo systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; preset: enabled)
     Active: active (running) since Mon 2024-11-25 14:08:11 CET; 6h ago
       Docs: man:firewalld(1)
   Main PID: 823 (firewalld)
      Tasks: 2 (limit: 48902)
     Memory: 42.9M
        CPU: 814ms
     CGroup: /system.slice/firewalld.service
             └─823 /usr/bin/python3 -s /usr/sbin/firewalld --nofork --nopid

Nov 25 14:08:10 localhost systemd[1]: Starting firewalld - dynamic firewall daemon...
Nov 25 14:08:11 localhost systemd[1]: Started firewalld - dynamic firewall daemon.
[root@localhost audit]# sudo firewall-cmd --add-port=80/tcp --permanent
success
[root@localhost audit]# sudo firewall-cmd --add-port=22/tcp --permanent
success
[root@localhost audit]# sudo firewall-cmd --add-port=443/tcp --permanent
success
[root@localhost audit]# sudo firewall-cmd --set-default-zone=drop
success
[root@localhost audit]# sudo firewall-cmd --reload
success
```
2. Galere aussi
3. Pour restreindre l'acces ssh a un certain range d'ip j'ai ajouté la ligne AllowUsers `*@192.168.1.*` dans `/etc/ssh/sshd_config`
