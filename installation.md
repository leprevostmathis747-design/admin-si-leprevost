# Fiche d'installation : TP1 (SRV-DC01 et SRV-WEB01)

Cette fiche permet de reproduire le SI décrit dans `inventaire.md`. Les commandes sont celles qui ont été réellement passées.

## 0. Écarts par rapport au sujet

| Sujet | Réalisé | Raison |
|---|---|---|
| VMware / VirtualBox | UTM pour les deux VM | Mac Apple Silicon : aucun ISO Windows Server ARM64 n'a pu démarrer sous VMware Fusion |
| Passerelle `10.10.0.2` | `10.10.0.240` | UTM place la passerelle et le DNS au début de sa plage DHCP |
| DHCP de l'hyperviseur désactivé | DHCP UTM limité à `.240`-`.250` | UTM ne permet pas de le désactiver ; l'étendue de SRV-DC01 est `.100`-`.200`, sans chevauchement |
| Carte `ens33` | `enp0s1` | Nom attribué par Debian avec la carte `virtio-net-pci` |
| Script `New-StructureAD.ps1` | Transféré depuis le Mac | Servi par un serveur HTTP temporaire (voir §3) |

## 1. Réseau UTM

Pour chaque VM : Réseau → mode **Réseau partagé**, réglages avancés :

- Réseau de l'invité : `10.10.0.0/24`
- Début DHCP : `10.10.0.240`, fin DHCP : `10.10.0.250`

## 2. SRV-DC01 : configuration initiale (PowerShell administrateur)

```powershell
Get-NetAdapter
Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Disabled
Remove-NetIPAddress -InterfaceAlias "Ethernet" -Confirm:$false -ErrorAction SilentlyContinue
Remove-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix 0.0.0.0/0 -Confirm:$false -ErrorAction SilentlyContinue
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.10.0.10 -PrefixLength 24 -DefaultGateway 10.10.0.240
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1,8.8.8.8
Rename-Computer -NewName SRV-DC01 -Restart
```

Contrôle : `hostname`, `Get-NetIPConfiguration`, `ping 8.8.8.8`.

## 3. SRV-DC01 : Active Directory, DNS, DHCP

```powershell
Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
Install-ADDSForest -DomainName "pme.lan" -DomainNetbiosName "PME" -InstallDns -SafeModeAdministratorPassword (Read-Host -AsSecureString "Mot de passe DSRM") -Force
```

Après redémarrage (connexion `PME\Administrateur`) :

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
Add-DnsServerForwarder -IPAddress 8.8.8.8
Resolve-DnsName debian.org

Install-WindowsFeature DHCP -IncludeManagementTools
Add-DhcpServerInDC -DnsName SRV-DC01.pme.lan -IPAddress 10.10.0.10
Add-DhcpServerv4Scope -Name "LAN-PME" -StartRange 10.10.0.100 -EndRange 10.10.0.200 -SubnetMask 255.255.255.0 -State Active
Set-DhcpServerv4OptionValue -ScopeId 10.10.0.0 -Router 10.10.0.240 -DnsServer 10.10.0.10 -DnsDomain pme.lan
Get-DhcpServerv4OptionValue -ScopeId 10.10.0.0
```

### Structure de l'annuaire

Le script `New-StructureAD.ps1` fourni par l'intervenant est servi depuis le Mac :

```bash
# Mac, dans le dossier contenant le script (python3 indisponible sans outils Xcode, donc Ruby)
ruby -run -e httpd . -p 8000
```

```powershell
# SRV-DC01
New-Item -ItemType Directory -Path C:\Scripts -Force
Invoke-WebRequest http://10.10.0.240:8000/New-StructureAD.ps1 -OutFile C:\Scripts\New-StructureAD.ps1
cd C:\Scripts
Unblock-File .\New-StructureAD.ps1
powershell -ExecutionPolicy Bypass -File .\New-StructureAD.ps1
```

Le serveur HTTP du Mac est arrêté (Ctrl+C) après le transfert. Le script est relancé une seconde fois : il n'affiche que des `[EXISTE]` (idempotence). Les mots de passe des comptes sont réinitialisés avec :

```powershell
Set-ADAccountPassword -Identity a.martin -Reset -NewPassword (Read-Host -AsSecureString "Mot de passe a.martin")
Set-ADAccountPassword -Identity b.durand -Reset -NewPassword (Read-Host -AsSecureString "Mot de passe b.durand")
```

## 4. SRV-WEB01 : installation et réseau

Installation Debian 13 (netinst arm64), mode texte : nom `srv-web01`, domaine `pme.lan`, root sans mot de passe, utilisateur `sysadmin` (sudo), partitionnement assisté, paquets « serveur SSH » et « utilitaires usuels » uniquement.

IP fixe dans `/etc/network/interfaces` :

```
auto enp0s1
iface enp0s1 inet static
    address 10.10.0.20/24
    gateway 10.10.0.240
iface enp0s1 inet6 auto
```

DNS rendu permanent (dhcpcd réécrit `resolv.conf` à chaque démarrage) :

```bash
printf 'search pme.lan\nnameserver 10.10.0.10\n' | sudo tee /etc/resolv.conf.head
printf 'search pme.lan\nnameserver 10.10.0.10\n' | sudo tee /etc/resolv.conf
```

Contrôle après redémarrage : `hostname -f` (`srv-web01.pme.lan`), `getent ahostsv4 srv-dc01.pme.lan` (`10.10.0.10`), `ping -c 2 debian.org`.

## 5. SRV-WEB01 : jonction au domaine

```bash
# Heure synchronisée sur le DC (Kerberos tolère 5 minutes d'écart)
sudo mkdir -p /etc/systemd/timesyncd.conf.d
printf "[Time]\nNTP=srv-dc01.pme.lan\n" | sudo tee /etc/systemd/timesyncd.conf.d/pme.conf
sudo systemctl restart systemd-timesyncd

sudo apt update
sudo apt install -y realmd sssd sssd-tools adcli libnss-sss libpam-sss krb5-user samba-common-bin packagekit
sudo realm discover pme.lan
sudo realm join -U Administrateur --computer-ou="OU=Serveurs,OU=PME,DC=pme,DC=lan" pme.lan
sudo realm list
id a.martin@pme.lan
id b.durand@pme.lan

# Dossier personnel automatique et sudo pour le groupe de l'annuaire
sudo pam-auth-update --enable mkhomedir
echo '%gg_admins_linux@pme.lan ALL=(ALL:ALL) ALL' | sudo tee /etc/sudoers.d/admins-linux
sudo chmod 440 /etc/sudoers.d/admins-linux
sudo visudo -c
```

Tests depuis le Mac : `ssh a.martin@pme.lan@10.10.0.20` puis `sudo whoami` (`root`) ; `ssh b.durand@pme.lan@10.10.0.20` puis `sudo whoami` (refusé).

## 6. SRV-WEB01 : SSH par clé

```bash
# Mac
ssh-keygen -t ed25519 -C "mathis@mac"
ssh-copy-id sysadmin@10.10.0.20
ssh sysadmin@10.10.0.20          # sans mot de passe
```

```bash
# SRV-WEB01 (session de secours gardée ouverte pendant le test)
printf 'PermitRootLogin no\nPubkeyAuthentication yes\n' | sudo tee /etc/ssh/sshd_config.d/durcissement.conf
sudo sshd -t && sudo systemctl reload ssh
sudo sshd -T | grep -Ei 'permitrootlogin|pubkeyauthentication'
```

`PasswordAuthentication` n'est pas désactivé : les comptes de l'annuaire se connectent encore par mot de passe.

## 7. SRV-WEB01 : nginx et pare-feu

```bash
sudo apt install -y nginx ufw curl
echo "<h1>SRV-WEB01 - pme.lan</h1>" | sudo tee /var/www/html/index.html

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH           # avant « enable », sinon la session SSH est coupée
sudo ufw allow 'Nginx HTTP'
sudo ufw enable
sudo ufw status verbose
```

Contrôles : `curl -I http://localhost` et `curl -I http://10.10.0.20` (`HTTP/1.1 200 OK`), page visible dans un navigateur.

## 8. Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|---|---|---|
| Aucun ISO Windows Server compatible avec Fusion | Pas d'ISO officiel ARM64 | Windows Server dans UTM, et Debian aussi (réseau partagé commun) |
| Passerelle différente du sujet | UTM utilise `10.10.0.240` | Relevée avec `ipconfig /all`, utilisée partout à la place de `.2` |
| `A positional parameter cannot be found` | Espace entre `-` et le nom du paramètre | Commande retapée, tiret collé au mot |
| `Set-DhcpServerv4...` non reconnu | Chiffre `0` tapé à la place de la lettre `O` d'`Option` | Complétion avec Tab |
| `Install-WindowsFeature DHCP` en `InvalidArgs` | Faute de saisie dans la première tentative | Commande retapée, rôle installé |
| `python3` inutilisable sur le Mac | Outils de développement Xcode absents | Serveur HTTP avec `ruby -run -e httpd` |
| `Invoke-WebRequest` impossible à joindre | Serveur HTTP du Mac non démarré | Serveur lancé, `Test-NetConnection 10.10.0.240 -Port 8000` |
| `resolv.conf` réécrit au redémarrage | dhcpcd regénère le fichier | `/etc/resolv.conf.head` |
| `getent` renvoie l'IPv6 du DC | Debian préfère l'IPv6 | `getent ahostsv4` pour voir `10.10.0.10` |
| `ssh-keygen` lancé sur SRV-WEB01 | Mauvaise machine | Annulation, clé générée sur le Mac |
| Mot de passe `b.durand` refusé | Saisie erronée | Mot de passe réinitialisé avec `Set-ADAccountPassword` |
