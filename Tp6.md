# TP – Mise en place d'un serveur OpenVPN sur Ubuntu Server

## 0. Contexte et objectifs

Dans ce TP, j’ai installé et configuré un serveur OpenVPN sur une VM Ubuntu Server LTS, mis en place une infrastructure de certificats (PKI) avec Easy‑RSA, activé le routage/NAT pour donner accès à Internet aux clients VPN, et généré un profil client fonctionnel.

---

## 1. Mise en place de la VM et préparation du système

- VM : Ubuntu Server LTS
- Mode réseau : NAT
- Accès distant : SSH

### Commandes

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install openvpn easy-rsa iptables-persistent -y
```

## Partie 1 

Infrastructure easy-RSA
```bash
sudo mkdir -p /etc/openvpn/easy-rsa
sudo cp -r /usr/share/easy-rsa/* /etc/openvpn/easy-rsa/
cd /etc/openvpn/easy-rsa/

sudo ./easyrsa init-pki
sudo ./easyrsa build-ca
```

Générations des certificats et des clés:

```bash
# Certificat serveur
sudo ./easyrsa gen-req server nopass
sudo ./easyrsa sign-req server server

# Certificat client
sudo ./easyrsa gen-req client1 nopass
sudo ./easyrsa sign-req client client1

# Paramètres Diffie-Hellman
sudo ./easyrsa gen-dh

# Clé TLS supplémentaire (tls-auth)
sudo openvpn --genkey secret pki/ta.key
```

### Où Easy‑RSA crée-t-il ses fichiers ?
Tous les fichiers sont créés dans le répertoire pki/ sous le dossier où l’on a lancé init-pki, ici :
/etc/openvpn/easy-rsa/pki/.

### Que contient le dossier pki/ ?

ca.crt, ca.key : certificat et clé de la CA

issued/ : certificats signés (server.crt, client1.crt, ...)

private/ : clés privées (server.key, client1.key, ...)

reqs/ : requêtes de certificats (CSR)

dh.pem : paramètres Diffie‑Hellman

éventuellement ta.key (clé TLS statique)

### Quelle est la différence entre gen-req et sign-req ?

gen-req génère une clé privée + une requête de certificat (CSR).

sign-req prend cette requête et la signe avec la clé de la CA pour produire un certificat valide.

### Que se passe-t-il si vous oubliez de signer un certificat ?
On ne dispose que d’une requête (CSR) et pas d’un certificat signé. Le serveur ou le client ne pourront pas l’utiliser pour s’authentifier dans le VPN.

## Partie 2
### 1. Copies des fichiers nécéssaire
sudo cp /etc/openvpn/easy-rsa/pki/ca.crt /etc/openvpn/
sudo cp /etc/openvpn/easy-rsa/pki/issued/server.crt /etc/openvpn/
sudo cp /etc/openvpn/easy-rsa/pki/private/server.key /etc/openvpn/
sudo cp /etc/openvpn/easy-rsa/pki/dh.pem /etc/openvpn/
sudo cp /etc/openvpn/easy-rsa/pki/ta.key /etc/openvpn/

### 2. Fichier de configuration
port 1194
proto udp
dev tun

ca /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key /etc/openvpn/server.key
dh /etc/openvpn/dh.pem
tls-auth /etc/openvpn/ta.key 0

server 10.8.0.0 255.255.255.0

push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"

keepalive 10 120
persist-key
persist-tun
user nobody
group nogroup
cipher AES-256-GCM
verb 3

### Questions
#### Que signifie dev tun ?
dev tun indique qu’OpenVPN utilisera une interface virtuelle de type TUN, qui travaille au niveau 3 (IP, routage), contrairement à tap qui est de type Ethernet (niveau 2).

#### Quelle est la différence entre UDP et TCP pour un VPN ?

UDP est plus léger et plus rapide, avec moins de surcharge, adapté au VPN pour de bonnes performances.

TCP gère la fiabilité (retransmissions, contrôle de flux) mais peut être plus lent et provoquer des effets « tunnel dans tunnel » si le trafic à l’intérieur est déjà en TCP.

#### Quelle plage IP choisir pour le VPN ? Pourquoi ?
On choisit une plage privée dédiée (par exemple 10.8.0.0/24) qui ne se chevauche pas avec les autres réseaux locaux. Cela évite les conflits de routes entre le LAN et le réseau VPN.

### Activer le forwarding IP
Fichier /etc/sysctl.conf :
net.ipv4.ip_forward=1

### Regles NAT
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o ens33 -j MASQUERADE
sudo netfilter-persistent save

### Questions
#### Où se configure le paramètre ip_forward ?
Dans le fichier /etc/sysctl.conf (et il est visible dans /proc/sys/net/ipv4/ip_forward).

#### Quelle commande permet d’afficher les règles NAT actuelles ?

```bash
sudo iptables -t nat -L -n -v
```
#### Pourquoi faut-il « masquerader » le réseau VPN ?
Le masquerading fait que toutes les IP du réseau VPN apparaissent sur Internet avec l’adresse de la machine passerelle. Les machines extérieures n’ont pas besoin de route vers 10.8.0.0/24, elles voient seulement l’IP publique de la VM.

## Démarrage et analyse du service
Commandes système
```bash
sudo systemctl start openvpn-server@server
sudo systemctl enable openvpn-server@server
sudo systemctl status openvpn-server@server

```

En cas de problème :
```bash
sudo journalctl -u openvpn-server@server
```

### Questions
#### Quelle commande permet d’afficher les logs système d’un service ?
journalctl -u nomduservice.

#### Quelle est la différence entre status et journalctl ?

systemctl status affiche l’état du service (actif/inactif) avec quelques dernières lignes de log.

journalctl -u affiche l’historique complet des logs du service, utile pour diagnostiquer des erreurs plus anciennes.

#### Les chemins vers les certificats sont-ils corrects ?
Dans server.conf, les chemins doivent correspondre aux fichiers présents dans /etc/openvpn/ (par exemple ca /etc/openvpn/ca.crt, cert /etc/openvpn/server.crt, etc.). Une erreur de chemin empêche le service de démarrer.

## Partie 3 – Création du profil client

### Fichier client1.ovpn
Sur le serveur :

```bash
mkdir -p ~/client-configs
nano ~/client-configs/client1.ovpn
```

### Questions
#### Comment intégrer un certificat directement dans un fichier .ovpn ?
On utilise des balises comme <ca> ... </ca>, <cert> ... </cert>, <key> ... </key>, et on colle le contenu des fichiers (certificats/clé) entre ces balises.

#### Pourquoi la clé privée ne doit-elle jamais être partagée publiquement ?
La clé privée permet de prouver l’identité du client et de déchiffrer une partie des échanges. Si quelqu’un obtient cette clé, il peut se connecter au VPN à la place de l’utilisateur ou espionner son trafic.

