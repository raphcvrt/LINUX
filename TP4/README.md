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
1. le script ressemble a ça et il a été entièrement fait par ChatGPT car je ne connais pas bash 
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
