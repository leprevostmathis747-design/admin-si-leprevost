# Inventaire du SI : TP1 (pme.lan)

Auteur : _à compléter_ · Date : 20/10/2026

## 1. Environnement de virtualisation

| Élément | Valeur |
|---|---|
| Machine hôte | Mac Apple Silicon (M5), macOS |
| Hyperviseur | UTM (les deux VM) |
| Réseau des VM | Réseau partagé UTM, `10.10.0.0/24` |
| Passerelle (hôte UTM) | `10.10.0.240` |
| DHCP UTM (non désactivable) | `10.10.0.240` à `10.10.0.250` |

## 2. Machines

| Machine | Rôle | OS | Adresse | Ressources |
|---|---|---|---|---|
| SRV-DC01 | Contrôleur de domaine (AD DS, DNS, DHCP) | Windows Server _(version à compléter)_ | `10.10.0.10/24` (fixe) | _à compléter_ |
| SRV-WEB01 | Serveur web joint au domaine | Debian 13 (arm64, noyau 6.12) | `10.10.0.20/24` (fixe) | 1 Go RAM, 1 vCPU, 20 Go |

## 3. Réseau

| Paramètre | SRV-DC01 | SRV-WEB01 |
|---|---|---|
| Interface | `Ethernet` | `enp0s1` |
| Adresse | `10.10.0.10/24` | `10.10.0.20/24` |
| Passerelle | `10.10.0.240` | `10.10.0.240` |
| DNS | `127.0.0.1` (redirecteur `8.8.8.8`) | `10.10.0.10` |
| Domaine de recherche | `pme.lan` | `pme.lan` |

## 4. Annuaire Active Directory

| Élément | Valeur |
|---|---|
| Domaine | `pme.lan` (NetBIOS : `PME`) |
| Unités d'organisation | `PME`, `PME/Utilisateurs`, `PME/Groupes`, `PME/Serveurs` |
| Groupes | `GG_Admins_Linux`, `GG_Utilisateurs` |
| Compte ordinateur | `CN=SRV-WEB01,OU=Serveurs,OU=PME,DC=pme,DC=lan` |

| Compte | Nom | Groupes | Droit sudo sur SRV-WEB01 |
|---|---|---|---|
| `a.martin` | Alice Martin | `GG_Admins_Linux`, `GG_Utilisateurs` | Oui |
| `b.durand` | Bob Durand | `GG_Utilisateurs` | Non |

Les mots de passe ne figurent pas dans ce dépôt.

## 5. Services

| Service | Machine | Détail |
|---|---|---|
| DHCP | SRV-DC01 | Étendue `LAN-PME` : `10.10.0.100` à `10.10.0.200`, bail de 8 jours |
| Options DHCP | SRV-DC01 | 003 routeur `10.10.0.240`, 006 DNS `10.10.0.10`, 015 domaine `pme.lan` |
| DNS | SRV-DC01 | Zone `pme.lan`, redirecteur `8.8.8.8` |
| NTP | SRV-WEB01 | Synchronisé sur `srv-dc01.pme.lan` (systemd-timesyncd) |
| Web | SRV-WEB01 | nginx, port 80, page `/var/www/html/index.html` |
| SSH | SRV-WEB01 | Port 22, clé ed25519, `PermitRootLogin no` |
| Pare-feu | SRV-WEB01 | ufw : entrant refusé par défaut, 22/tcp et 80/tcp autorisés |

## 6. Vérifications de fin de TP

| Vérification | Résultat |
|---|---|
| Bail DHCP distribué | `10.10.0.100` actif pour `srv-web01.pme.lan` |
| Annuaire | `a.martin` et `b.durand` actifs |
| Jonction au domaine | `pme.lan`, `configured: kerberos-member` |
| Droits par groupe | `a.martin` : `root` ; `b.durand` : refusé |
| SSH par clé | Connexion sans mot de passe |
| Service web | `HTTP/1.1 200 OK` sur `http://10.10.0.20` |
| Pare-feu | Actif, 22 et 80 autorisés |
