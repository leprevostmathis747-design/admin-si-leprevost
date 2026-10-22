# Fiche stockage : TP2 partie 1 (SRV-WEB01)

Auteur : Leprevost · Date : 22/10/2026

Objectif : séparer `/var` et `/home` du système sur des volumes LVM, et pouvoir les agrandir sans arrêt de service. Cette fiche complète `inventaire.md` et `installation.md` (TP1).

## 0. Écarts par rapport au sujet

| Sujet | Réalisé | Raison |
|---|---|---|
| Disques `sda` à `sde` | `vda` à `vde` | Disques virtio sous UTM : tous les noms du sujet sont remplacés (`sdb` → `vdb`, etc.) |
| VMware / VirtualBox | UTM | Voir `installation.md` |
| Instantané « avant-TP2 » | Clone UTM de la VM | UTM ne propose pas les instantanés du sujet, le clone joue le même rôle |

## 1. Disques de SRV-WEB01

| Disque | Taille | Usage | Quand |
|---|---|---|---|
| `vda` | 20 Go | Système (TP1), non modifié | n/a |
| `vdb` | 5 Go | LVM, groupe `vg_data` | TP2 mardi |
| `vdc` | 5 Go | LVM, groupe `vg_data` | TP2 mardi |
| `vdd` | 5 Go | RAID 1 `md0` (avec `vde`) | TP2 jeudi, laissé vierge |
| `vde` | 5 Go | RAID 1 `md0` (avec `vdd`) | TP2 jeudi, laissé vierge |

## 2. Étape 1 : paquets

```bash
lsblk -d -o NAME,SIZE,TYPE,MODEL
sudo apt update
sudo apt install -y parted lvm2 rsync
```

SRV-DC01 est allumé avant SRV-WEB01 : c'est le seul DNS de la machine, sans lui `apt` échoue.

## 3. Étape 2 : partitions GPT

Une partition par disque, alignée à 1 MiB, marquée `lvm` :

```bash
sudo parted -s /dev/vdb mklabel gpt mkpart lvm 1MiB 100% set 1 lvm on
sudo parted -s /dev/vdc mklabel gpt mkpart lvm 1MiB 100% set 1 lvm on
sudo parted /dev/vdb print
sudo parted /dev/vdc print
lsblk /dev/vdb /dev/vdc
```

Contrôle : `Partition Table: gpt`, partition 1 de 5367 MB commençant à 1049 kB, drapeau `lvm`.

## 4. Étape 3 : LVM et ext4

```bash
sudo pvcreate /dev/vdb1 /dev/vdc1
sudo vgcreate vg_data /dev/vdb1 /dev/vdc1
sudo lvcreate -n lv_var -L 4G vg_data
sudo lvcreate -n lv_home -L 3G vg_data
sudo pvs ; sudo vgs ; sudo lvs
sudo mkfs.ext4 -L var /dev/vg_data/lv_var
sudo mkfs.ext4 -L home /dev/vg_data/lv_home
sudo blkid /dev/vg_data/lv_var /dev/vg_data/lv_home
```

Contrôle : `vg_data` d'environ 10 Go avec 2,99 Go libres, `lv_var` 4,00 Go, `lv_home` 3,00 Go, tous deux en ext4.

## 5. Étape 4 : copie de /var et /home

Montage provisoire des nouveaux volumes, arrêt des services qui écrivent dans `/var`, puis copie en conservant propriétaires, droits, ACL, attributs étendus, liens durs et fichiers creux.

```bash
sudo mkdir -p /mnt/var /mnt/home
sudo mount /dev/vg_data/lv_var /mnt/var
sudo mount /dev/vg_data/lv_home /mnt/home
sudo systemctl stop nginx sssd
sudo systemctl stop cron apt-daily.timer apt-daily-upgrade.timer 2>/dev/null
findmnt /mnt/var ; findmnt /mnt/home

sudo rsync -aAXHS --numeric-ids --info=progress2 /var/ /mnt/var/ ; echo "Code retour : $?"
sudo rsync -aAXHS --numeric-ids --info=progress2 /home/ /mnt/home/ ; echo "Code retour : $?"
```

Les deux copies se terminent avec `Code retour : 0`.

Vérification par simulation (`-n`, rien n'est écrit) :

```bash
sudo rsync -aAXHSn --numeric-ids --itemize-changes /var/ /mnt/var/ | grep -v '^\.d' | head -20
sudo du -sh /var /mnt/var /home /mnt/home
sudo umount /mnt/var /mnt/home
```

| Résultat | Valeur |
|---|---|
| Lignes de la simulation | 3 lignes `>f..t` (`timesync/clock` et deux fichiers `.journal`), aucune `>f+++++++++` |
| `/var` / `/mnt/var` | 440M / 406M (journald réserve de l'espace d'avance) |
| `/home` / `/mnt/home` | 96K / 112K (le `lost+found` d'un ext4 neuf ajoute 16K) |

Points techniques :

- `-S` recrée les fichiers creux tels quels. C'est indispensable pour `/var/log/lastlog`, dont la taille apparente est énorme (UID de l'annuaire très élevés, environ 123 Go apparents pour quelques Ko réels). Sans `-S`, la copie remplirait `lv_var`.
- Le `/` final de `/var/` compte : sans lui on obtient `/mnt/var/var`.
- `--numeric-ids` : `sssd` est arrêté, les noms des comptes de l'annuaire ne se résolvent plus, on copie donc les numéros (UID et GID).

## 6. Étape 5 : fstab par UUID

```bash
sudo cp /etc/fstab /etc/fstab.avant-tp2
echo "UUID=$(sudo blkid -s UUID -o value /dev/vg_data/lv_var) /var ext4 defaults,nodev,nosuid 0 2" | sudo tee -a /etc/fstab
echo "UUID=$(sudo blkid -s UUID -o value /dev/vg_data/lv_home) /home ext4 defaults,nodev,nosuid 0 2" | sudo tee -a /etc/fstab
tail -3 /etc/fstab
sudo systemctl daemon-reload
sudo findmnt --verify
```

Résultat de `findmnt --verify` : 0 erreur d'analyse, 0 erreur, 1 alerte (lecteur CD vide `/media/cdrom0`, option `noauto`, sans conséquence).

### fstab final

```
UUID=2dedfc01-89db-4fe0-95af-56734b09f264 /          ext4  errors=remount-ro 0 1
UUID=7A61-D9E7                            /boot/efi  vfat  umask=0077        0 1
UUID=c8605766-f99d-4315-ac01-2e04d5a126b3 none       swap  sw                0 0
/dev/sr0                                  /media/cdrom0 udf,iso9660 user,noauto 0 0
UUID=2d84177d-b093-4419-9e13-d0802e80b51c /var       ext4  defaults,nodev,nosuid 0 2
UUID=89e672fe-72a6-4cd3-8aeb-b640ce924d3e /home      ext4  defaults,nodev,nosuid 0 2
```

## 7. Étape 6 : vérifications après redémarrage

```bash
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS /var
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS /home
df -h / /var /home
systemctl --failed
id a.martin@pme.lan
curl -I http://localhost
```

| Contrôle | Résultat |
|---|---|
| `/var` | `/dev/mapper/vg_data-lv_var`, ext4, `rw,nosuid,nodev,relatime` |
| `/home` | `/dev/mapper/vg_data-lv_home`, ext4, `rw,nosuid,nodev,relatime` |
| `df -h` | `/` 18G à 10 %, `/var` 3,9G à 12 %, `/home` 2,9G à 1 % |
| Annuaire | `a.martin` : UID `415201105`, groupes `gg_admins_linux` et `gg_utilisateurs` |
| Web | `HTTP/1.1 200 OK` |
| `systemctl --failed` | 3 sockets `sssd-nss`, `sssd-pac` et `sssd-pam` en échec |

Correction des sockets en échec (conflit hérité de la jonction du TP1) :

```bash
systemctl is-active sssd                      # active
sudo systemctl disable --now sssd-nss.socket sssd-pac.socket sssd-pam.socket
sudo systemctl reset-failed
systemctl --failed                            # 0 loaded units listed
id a.martin@pme.lan                           # l'annuaire répond toujours
```

### Test de `nosuid`

```bash
sudo cp /usr/bin/id /home/sysadmin/id-suid
sudo chmod u+s /home/sysadmin/id-suid
ls -l /home/sysadmin/id-suid        # -rwsr-xr-x 1 root root
/home/sysadmin/id-suid              # uid=1000(sysadmin) ..., sans euid=0
sudo rm /home/sysadmin/id-suid
```

Le bit SUID est bien posé, mais le programme s'exécute avec les droits de `sysadmin` seulement : `nosuid` empêche l'élévation de privilèges depuis `/home`.

### Agrandissement à chaud de /home

Écriture continue dans une session, agrandissement dans une autre :

```bash
# session 2 : une ligne par seconde dans /home
while true; do date >> ~/horloge.txt; sleep 1; done

# session 1
df -h /home
sudo lvextend -r -L +1G /dev/vg_data/lv_home
df -h /home
sudo vgs
```

| Mesure | Valeur |
|---|---|
| `lv_home` avant | 3,00 Go (`/home` : 2,9G) |
| `lv_home` après +1G | 4,00 Go (`/home` : 3,9G) |
| `vg_data` après +1G | 1,99 Go libres |

`lvextend -r` agrandit le volume et son ext4 en une seule commande, sur un système de fichiers monté (« on-line resizing »). Deux extensions de +100M ont ensuite été faites pendant la boucle d'écriture, ce qui amène `lv_home` à 4,20 Go (`/home` : 4,1G).

Preuve de la continuité : `dmesg -T | grep -i resized` date les redimensionnements à 09:53:26, 09:55:31 et 09:56:56. Le dernier tombe dans la plage de `horloge.txt` (38 lignes, de 09:56:40 environ à 09:57:17). Un contrôle `awk` ne détecte aucun écart de plus de 2 secondes entre deux lignes : l'écriture n'a jamais été interrompue.

## 8. État final

```
lsblk -f   (extrait)
vda3  ext4   /          15,1G libres (9 %)
vda2  vfat   /boot/efi
vda4  swap
vdb1  LVM2_member -> vg_data-lv_var   ext4 (var)  /var   3,2G libres (11 %)
vdc1  LVM2_member -> vg_data-lv_home  ext4 (home) /home  3,8G libres (0 %)
vdd, vde : vierges (RAID 1 de jeudi)

vgs : vg_data  2 PV  2 LV  VSize 9,99g  VFree <1,80g
```

## 9. Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|---|---|---|
| `parted` : « Could not stat device /dev/sdb » | Les disques virtio s'appellent `vdb`, pas `sdb` | `lsblk -d` avant toute commande, noms adaptés. La commande ratée n'a rien modifié |
| `lsblk -f` ne montre ni ext4 ni UUID pour `lv_home` | Retard d'affichage juste après `mkfs` | `sudo blkid` confirme `TYPE="ext4"` et `LABEL="home"` |
| 3 sockets `sssd-*` en échec après redémarrage | Conflit hérité de la jonction du TP1 | `disable --now` des sockets, `reset-failed` |
| `horloge.txt` introuvable après le test | Commande `ssh` collée avec la boucle : la boucle partait avant la connexion | Boucle relancée seule une fois connecté |
| Alerte `findmnt --verify` sur `/media/cdrom0` | Lecteur CD vide, option `noauto` | Alerte normale, ignorée |
