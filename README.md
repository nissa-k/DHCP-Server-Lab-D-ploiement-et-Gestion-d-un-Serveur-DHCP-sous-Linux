# DHCP Server Lab – Déploiement et Gestion d'un Serveur DHCP sous Linux

### TP3 – Mise en place d'un DHCP

**Autrice :** Karadag Nissa  
**Formation :** Bachelor 2 Informatique – Ynov Campus  
**Module :** Systèmes Linux  
**Date :** Janvier 2026

---

## Sommaire

1. [Objectif](#objectif)
2. [Infrastructure](#infrastructure)
3. [Configuration réseau du serveur](#configuration-réseau-du-serveur-srv-01)
4. [Installation du serveur DHCP](#installation-du-serveur-dhcp)
5. [Configuration du serveur DHCP](#configuration-du-serveur-dhcp)
6. [Démarrage et vérification](#démarrage-et-vérification-du-service)
7. [Configuration côté client](#configuration-côté-client)
8. [Script de maintenance des baux](#script-de-maintenance-des-baux-dhcp)
9. [Automatisation via crontab](#automatisation-via-crontab)
10. [Bilan](#bilan)

---

## Objectif

Déployer un serveur DHCP sous Linux afin d'automatiser l'attribution des adresses IP pour le réseau 192.168.50.0/24. Le TP couvre l'installation, la configuration, les tests côté client, la réservation d'adresse par MAC, la gestion des baux et l'automatisation de la maintenance.

---

## Infrastructure

| Machine | Role | Carte 1 | Carte 2 | IP LAN |
|---|---|---|---|---|
| SRV-01 (srv1) | Serveur DHCP | NAT (Internet) | Réseau interne intnet | 192.168.50.10/24 (statique) |
| SRV-01 (client) | Client DHCP | Réseau interne intnet | — | 192.168.50.50 (via réservation) |

**Parametres DHCP configurés :**

| Parametre | Valeur |
|---|---|
| Réseau | 192.168.50.0/24 |
| Plage DHCP | 192.168.50.100 – 192.168.50.200 |
| Passerelle | 192.168.50.1 |
| DNS | 8.8.8.8 |
| Bail par défaut | 600 secondes |
| Bail maximum | 7200 secondes |
| Réservation MAC | 08:00:27:b5:16:14 → 192.168.50.50 |

---

## Configuration réseau du serveur (SRV-01)

L'interface `enp0s3` est configurée en DHCP (accès Internet via NAT) et `enp0s8` reçoit une IP statique sur le réseau interne.

**Fichier `/etc/netplan/50-cloud-init.yaml` :**

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s3:
      dhcp4: true

    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.50.10/24
      routes:
        - to: default
          via: 192.168.50.1
      nameservers:
        addresses:
          - 8.8.8.8
```

```bash
sudo chmod 600 /etc/netplan/*.yaml
sudo netplan apply
ip a    # Vérification : enp0s8 → 192.168.50.10/24
```

---

## Installation du serveur DHCP

```bash
sudo apt update
sudo apt install isc-dhcp-server
```

---

## Configuration du serveur DHCP

### Interface d'écoute – `/etc/default/isc-dhcp-server`

```bash
sudo nano /etc/default/isc-dhcp-server
```

```
INTERFACESv4="enp0s8"
INTERFACESv6=""
```

### Configuration principale – `/etc/dhcp/dhcpd.conf`

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

```
authoritative;

default-lease-time 600;
max-lease-time 7200;

subnet 192.168.50.0 netmask 255.255.255.0 {
  range 192.168.50.100 192.168.50.200;
  option routers 192.168.50.1;
  option domain-name-servers 8.8.8.8;
  option broadcast-address 192.168.50.255;
}

host client-fixe {
  hardware ethernet 08:00:27:b5:16:14;
  fixed-address 192.168.50.50;
}
```

---

## Démarrage et vérification du service

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

Résultat attendu : `Active: active (running)`

```bash
sudo ss -ulnp | grep :67
# UNCONN 0  0  0.0.0.0:67  0.0.0.0:*  users:(("dhcpd",...))
```

Le service écoute correctement sur le port UDP 67 (port DHCP standard).

---

## Configuration côté client

### Récupération de l'adresse MAC

```bash
ip a
ip link
# enp0s3 → MAC : 08:00:27:b5:16:14
```

### Installation du client DHCP

```bash
sudo su
apt install isc-dhcp-client -y
```

### Activation de l'interface et netplan client

```bash
sudo ip link set enp0s3 up
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true

    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.50.10/24
```

```bash
sudo chmod 600 /etc/netplan/*.yaml
sudo netplan apply
```

### Forcer l'attribution DHCP

```bash
sudo dhclient -v enp0s3
```

Séquence observée :

```
DHCPDISCOVER on enp0s3 to 255.255.255.255 port 67
DHCPOFFER of 192.168.50.50 from 192.168.50.10
DHCPREQUEST for 192.168.50.50 on enp0s3
DHCPACK of 192.168.50.50 from 192.168.50.10
bound to 192.168.50.50 -- renewal in 242 seconds.
```

La réservation fonctionne : le client reçoit systématiquement l'IP 192.168.50.50 correspondant à sa MAC.

---

## Script de maintenance des baux DHCP

**Fichier `/usr/local/sbin/dhcp_leases_maintenance.sh` :**

```bash
LEASES="/var/lib/dhcp/dhcpd.leases"
ARCHIVE="/var/log/dhcp-leases-archive"
DATE=$(date +%F)

mkdir -p "$ARCHIVE"
cp "$LEASES" "$ARCHIVE/dhcpd.leases.$DATE"

# Supprimer les archives de plus de 30 jours
find "$ARCHIVE" -type f -mtime +30 -delete
```

```bash
sudo chmod +x /usr/local/sbin/dhcp_leases_maintenance.sh
sudo /usr/local/sbin/dhcp_leases_maintenance.sh
ls /var/log/dhcp-leases-archive/
# dhcpd.leases.2026-01-16
```

---

## Automatisation via crontab

```bash
sudo crontab -e
```

Ligne ajoutée à la fin du fichier :

```
0 2 * * * /usr/local/sbin/dhcp_leases_maintenance.sh
```

```bash
sudo crontab -l
# Vérification : tâche planifiée à 2h00 chaque nuit
```

Le script s'exécute automatiquement chaque nuit à 2h00. Il archive le fichier des baux DHCP et supprime les archives de plus de 30 jours, assurant une maintenance sans intervention manuelle.

---

## Bilan

| Etape | Action | Résultat |
|---|---|---|
| Config réseau serveur | enp0s8 → 192.168.50.10/24 statique | Validé |
| Installation DHCP | apt install isc-dhcp-server | Validé – service actif |
| Config interface | INTERFACESv4="enp0s8" | Validé |
| Config dhcpd.conf | Plage .100-.200, gateway, DNS | Validé |
| Réservation MAC | 08:00:27:b5:16:14 → .50 | Validé – DHCPACK reçu |
| Test client DHCP | dhclient -v enp0s3 | Validé – IP .50 attribuée |
| Script maintenance | Archivage baux + suppression >30j | Validé – archive créée |
| Automatisation cron | 0 2 * * * script.sh | Validé – crontab -l OK |

---

*Karadag Nissa — Bachelor 2 Informatique — Ynov Campus — 2026*
