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

# Définir les paramètres du volume
GROUPE_VOLUME="debian--vg"
VOLUME_LOGIQUE="root"
SNAPSHOT="root_snapshot"
TAILLE_SNAPSHOT="2G"  # Taille à ajuster entre 1G et 5G
POINT_DE_MONTAGE="/mnt/snapshot"

# Créer un snapshot du volume logique
echo "Création du snapshot..."
sudo lvcreate --size $TAILLE_SNAPSHOT --snapshot --name $SNAPSHOT /dev/$GROUPE_VOLUME/$VOLUME_LOGIQUE

# Vérification de la création du snapshot
if [ $? -eq 0 ]; then
    echo "Snapshot créé avec succès."
else
    echo "Erreur lors de la création du snapshot."
    exit 1
fi

# Créer le répertoire de montage
echo "Création du répertoire de montage..."
sudo mkdir -p $POINT_DE_MONTAGE

# Monter le snapshot
echo "Montage du snapshot..."
sudo mount /dev/$GROUPE_VOLUME/$SNAPSHOT $POINT_DE_MONTAGE

# Vérification du montage
if mount | grep $POINT_DE_MONTAGE > /dev/null; then
    echo "Snapshot monté avec succès."
else
    echo "Erreur lors du montage."
    exit 1
fi

# Faire une sauvegarde ici si nécessaire

# Démonter et supprimer le snapshot après utilisation
echo "Démontage et suppression du snapshot..."
sudo umount $POINT_DE_MONTAGE
sudo lvremove -f /dev/$GROUPE_VOLUME/$SNAPSHOT

echo "Snapshot supprimé."
```

je configure le crontab pour qu'il exécute le script tout les dimanche à minuit

```
0 0 * * 0 /home/paph/script_backup.sh
```
