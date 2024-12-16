je commence par me créer un user pour eviter de tout faire en root

```sh
root@debian:/etc/nginx# sudo adduser paph
root@debian:/etc/nginx# usermod -aG sudo paph
```

je desactive le root pour le ssh dans ``

```sh
root@debian:/etc/nginx#/etc/ssh/sshd_config
root@debian:/etc/nginx#sudo systemctl reload sshd
```

j'installe `fail2ban` pour me proteger des bruteforce

Je modifie `/etc/fail2ban/jail.local` et j'ajoute :

```shell
[sshd]
enabled = true
port = ssh
logpath = /var/log/secure
maxretry = 5

```

```shell
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

```
paph@debian:~$ sudo apt install ufw -y
paph@debian:~$ sudo ufw allow ssh
paph@debian:~$ sudo ufw allow 80/tcp
paph@debian:~$ sudo ufw allow 443/tcp
paph@debian:~$ sudo ufw default deny incoming
paph@debian:~$ sudo ufw enable
```

je liste les disques

```
paph@debian:~$ lsblk
NAME                  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                     8:0    0    5G  0 disk 
|-sda1                  8:1    0  487M  0 part /boot
|-sda2                  8:2    0    1K  0 part 
`-sda5                  8:5    0  4.5G  0 part 
  |-debian--vg-root   254:0    0    4G  0 lvm  /
  `-debian--vg-swap_1 254:1    0  512M  0 lvm  [SWAP]
sdb                     8:16   0    2G  0 disk 
```

je crée un script qui backup le disque principal montage pour vérifier le snapshot (ChatGPT)

```
#!/bin/bash

# Variables
BACKUP_DIR="/mnt/snapshot_backup"
SOURCE_DIR="/"
DATE=$(date +%Y%m%d)
ARCHIVE_NAME="backup_root_${DATE}.tar.gz"

# Vérifier si le répertoire de sauvegarde existe, sinon le créer
if [ ! -d "$BACKUP_DIR" ]; then
    echo "Le répertoire $BACKUP_DIR n'existe pas. Création en cours..."
    mkdir -p "$BACKUP_DIR"
fi

# Créer une archive complète
echo "Création de l'archive complète dans $BACKUP_DIR/$ARCHIVE_NAME..."
tar -czf "$BACKUP_DIR/$ARCHIVE_NAME" "$SOURCE_DIR"
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

je configure le crontab pour qu'il exécute le script tout les dimanche à minuit

```
0 0 * * 0 /home/paph/script_backup.sh
```
