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

# Configuration
BACKUP_DIR="/mnt/snapshot_backup"  # Emplacement de sauvegarde
SOURCE_DIR="/"                     # Répertoire source à sauvegarder
DATE=$(date +%Y%m%d)               # Date du jour pour nommer les archives
SNAPSHOT_NAME="backup_${DATE}.tar.gz"  # Nom de l'archive
EXCLUDE_FILE="/home/paph/exclude_list.txt"  # Fichier d'exclusion

# Préparation
echo "Préparation de la sauvegarde..."
sudo mkdir -p "$BACKUP_DIR"
if ! sudo mount /dev/sdb1 "$BACKUP_DIR"; then
    echo "Erreur : Impossible de monter /dev/sdb1. Vérifiez votre disque." >&2
    exit 1
fi

# Fichier d'exclusion
cat <<EOF > "$EXCLUDE_FILE"
/proc
/sys
/dev
/tmp
/run
/media
/mnt
/lost+found
$BACKUP_DIR
EOF
echo "Fichier d'exclusion créé : $EXCLUDE_FILE"

# Création de la sauvegarde
echo "Création de l'archive dans $BACKUP_DIR/$SNAPSHOT_NAME..."
if ! sudo tar --exclude-from="$EXCLUDE_FILE" -czf "$BACKUP_DIR/$SNAPSHOT_NAME" "$SOURCE_DIR"; then
    echo "Erreur : Échec de la création de l'archive." >&2
    sudo umount "$BACKUP_DIR"
    rm -f "$EXCLUDE_FILE"
    exit 1
fi
echo "Sauvegarde terminée avec succès : $BACKUP_DIR/$SNAPSHOT_NAME."

# Rotation des sauvegardes
echo "Nettoyage des anciennes sauvegardes (conservation des 5 plus récentes)..."
sudo ls -tp "$BACKUP_DIR" | grep -v '/$' | tail -n +6 | xargs -d '\n' -r sudo rm --
echo "Rotation des sauvegardes terminée."

# Nettoyage final
echo "Démontage du disque et suppression des fichiers temporaires..."
sudo umount "$BACKUP_DIR"
rm -f "$EXCLUDE_FILE"

echo "Processus de sauvegarde terminé avec succès."

```

je configure le crontab pour qu'il exécute le script tout les dimanche à minuit

```
0 0 * * 0 /home/paph/script_backup.sh
```
