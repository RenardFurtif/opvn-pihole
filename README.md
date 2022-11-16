# Installer OPVN incluant un PiHole sur Ubuntu
## _Une connnexion chiffreé et une suppression de pub !_

OpenVPN est un logiciel libre permettant de créer un réseau privé virtuel VPN. 
Le Pi-hole est un serveur DNS qui protège vos appareils des contenus indésirables sans installer de logiciel côté client.

- Difficulté : ★☆☆☆☆
- Puissance matérielle nécessaire : ★☆☆☆☆
- Temps : 20-45 minutes

## Prérequis matériels

- Un ordinateur qui devra rester constamment allumé ou un serveur dédié
- Une connexion internet stable et puissance côté serveur
- Un accès au shell de votre machine 

Pour ma part, j'utilise un serveur dédiée de chez OVH sous Ubuntu Server 20.04 LTS "Focal Fossa". Une puissance réseaux de 100 Mb/s en montant et en descendant (Je n'ai jamais eu de soucis de ralentissement)

## Installation
### Installation d'OpenVPN

Tout d'abord, nous allons mettre à jour nos depots puis mettre à jour la machine
```sh
sudo apt update && apt upgrade
```
Création d'un dossier pour stocker tout les fichiers que nous avons besoin pour installation et qui nous serviront pour la suite (Personnellement, je l'ai créé dans mon répétoire d'utilisateur).
```sh
mkdir OPVN_PiHole
```
Nous allons nous rendre de ce dossier
```sh
cd OPVN_PiHole
```
Nous allons télécharger OpenVPN, le rendre executable puis lancer l'installation
```sh
wget https://raw.githubusercontent.com/RenardFurtif/opvn-pihole/main/openvpn-install.sh
chmod 755 openvpn-install.sh
./openvpn-install.sh
```
En tapant les commandes ci dessus, nous entrons dans la face d'installation. L'installateur d'OpenVPN vous réclamera quelle informations. 

L'installateur nous demande quel protocol nous voulons utiliser pour le tunnel VPN.

Choisir l'option "1" donc le protocol UDP
```sh
Welcome to this quick OpenVPN "road warrior" installer

I need to ask you a few questions before starting the setup
You can leave the default options and just press enter if you are ok with them

First I need to know the IPv4 address of the network interface you want OpenVPN
listening to.
IP address: 10.8.0.1

Which protocol do you want for OpenVPN connections?
   1) UDP (recommended)
   2) TCP
Protocol [1-2]: 1
```
Ensuite l'instalateur nous demande un port pour le tunnel VPN (Par défaut : 1194)

Choisir le port "1194"
```sh
What port do you want OpenVPN listening to?
Port: 1194
```
Puis l'installateur nous demande de choisir un serveur DNS (Par défaut :  Current system resolvers)

Choisir l'option "1" donc le serveur DNS que vous utilisé actuellement sur votre machine. Vous pouvez choisir une autre option si un de vos serveur DNS préferer se trouvedans cet liste.
```sh
Which DNS do you want to use with the VPN?
   1) Current system resolvers
   2) Google
   3) OpenDNS
   4) NTT
   5) Hurricane Electric
   6) Verisign
DNS [1-6]: 1
```
Enfin, nous devons crée un nom pour une configuration client OpenVPN

Entrez le nom "PiHole", Vous pouvez changer celui-ci à votre guise. 
```sh
Finally, tell me your name for the client certificate
Please, use one word only, no special characters
Client name: pihole
Okay, that was all I needed. We are ready to setup your OpenVPN server now
Press any key to continue...
```
Bravo ! L'installation OpenVPN est terminé.
```sh
Finished!

Your client configuration is available at /Ubuntu/pihole.ovpn
If you want to add more clients, you simply need to run this script again!
```

### Installation de PiHole
Toujours dans le dossier "OPVN_PiHole"

Nous allons télécharger PiHole et lancer l'installation 
```sh
wget https://raw.githubusercontent.com/RenardFurtif/opvn-pihole/main/pihole-install.sh
sudo bash pihole-install.sh
```
choisissez tun0 comme interface et 10.8.0.1/24 comme adresse IP. Vous pouvez accepter le reste des valeurs par défaut

Bravo ! L'installation de PiHole est terminé.

## Configuration 
### Configuration d'OpenVPN

Tout d'abord, nous devons modifier le fichier config d'OPVN.
```sh
sudo nano /etc/openvpn/server/server.conf
```
Ensuite changer cette ligne 
```sh
push "dhcp-option DNS 8.8.8.8"
```
Par cela 
```sh
push "dhcp-option DNS 10.8.0.1"
```
Cette ligne indique aux clients se connectant au VPN qu'ils doivent utiliser Pi-hole comme serveur DNS primaire.

Maintenant, nous devons redémarrer le VPN. Si cette commande ne marche pas redémarrer entièrement la machine
```sh
systemctl restart openvpn-server@server
```
Bravo ! La configuration d'OpenVPN est terminé
### Configuration du pare-feu de PiHole 

Cette étape est la plud importante, c'est elle qui va permet de bloquer les publicité.

Ces commandes autoriseront le DNS et le HTTP nécessaires à la résolution de noms de domaines (en utilisant Pi-hole comme serveur dns) et l'accès à l'interface Web
```sh
iptables -A INPUT -i tun0 -p tcp --destination-port 53 -j ACCEPT
iptables -A INPUT -i tun0 -p udp --destination-port 53 -j ACCEPT
iptables -A INPUT -i tun0 -p tcp --destination-port 80 -j ACCEPT
```
Vous allons activer l'accès SSH et VPN depuis les ip publics et locales.
```sh
iptables -A INPUT -p tcp --destination-port 22 -j ACCEPT
iptables -A INPUT -p tcp --destination-port 1194 -j ACCEPT
iptables -A INPUT -p udp --destination-port 1194 -j ACCEPT
```
Le paramètre crucial suivant est d'autoriser TCP/IP à effectuer des "three-wayhandshakes" :

```sh
iptables -I INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
```
 nous allons autoriser tout trafic de bouclage, c'est-à-dire que le serveur est autorisé à se parler à lui-même sans aucune limitation en utilisant 127.0.0.0.
 
 ```sh
iptables -I INPUT -i lo -j ACCEPT
```

Puis, nous voulons rejeter l'accès provenant de n'importe où (c'est-à-dire si aucune règle ne correspond alors nous rejetons le traffic).
 ```sh
iptables -P INPUT DROP
```
Nous allons également en profiter pour bloquer les publicités HTTPS afin d'améliorer le blocage des publicités qui sont chargées via HTTPS et également traiter QUIC.
 ```sh
iptables -A INPUT -p udp --dport 80 -j REJECT --reject-with icmp-port-unreachable
iptables -A INPUT -p tcp --dport 443 -j REJECT --reject-with tcp-reset
iptables -A INPUT -p udp --dport 443 -j REJECT --reject-with icmp-port-unreachable
```
Point de verification de l'iptables, pour verifier si vous n'avez pas obligez de règles. 
Nous allons afficher la liste des règles du pare-feu

```sh
iptables -L --line-numbers
```
Pour mon cas j'ai cette reponse. Si tout s'est bien passer vous avez le resultat
```sh
Chain INPUT (policy DROP)
num  target     prot opt source               destination
1    ACCEPT     all  --  anywhere             anywhere
2    ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
3    ACCEPT     all  --  anywhere             anywhere
4    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:domain
5    ACCEPT     udp  --  anywhere             anywhere             udp dpt:domain
6    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http
7    ACCEPT     udp  --  anywhere             anywhere             udp dpt:80
8    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
9    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:openvpn
10   ACCEPT     udp  --  anywhere             anywhere             udp dpt:openvpn
11   ACCEPT     tcp  --  10.8.0.0/24          anywhere             tcp dpt:domain
12   ACCEPT     udp  --  10.8.0.0/24          anywhere             udp dpt:domain
13   ACCEPT     tcp  --  10.8.0.0/24          anywhere             tcp dpt:http
14   ACCEPT     udp  --  10.8.0.0/24          anywhere             udp dpt:80
15   ACCEPT     tcp  --  10.8.0.0/24          anywhere             tcp dpt:domain
16   ACCEPT     tcp  --  10.8.0.0/24          anywhere             tcp dpt:http
17   ACCEPT     udp  --  10.8.0.0/24          anywhere             udp dpt:domain
18   ACCEPT     udp  --  10.8.0.0/24          anywhere             udp dpt:80
19   REJECT     tcp  --  anywhere             anywhere             tcp dpt:https reject-with icmp-port-unreachable

Chain FORWARD (policy ACCEPT)
num  target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination
```
Si tout semble bien s'être bien passer, nous allons enregistrer nos règles afin de pouvoir y apporter des modifications plus tard.
```sh
iptables-save > /etc/pihole/rules.v4
```
Bravo ! La configuration du pare-feu de PiHole est terminé

## Ajour d'un client au VPN
Toujours dans le dossier "OPVN_PiHole"
Maintenant que tout fonctionne, il faut créé un nouveau utilisateur dans la configuration de notre VPN afin de pouvoir si connecter.

Nous allons lancer installateur OpenVPN qui est devenu une sorte de boite à outil OpenVPN
```sh
bash openvpn-install.sh
```
OpenVPN nous dit que le serveur est bien installer. Nous allons choisir l'option "1" donc l'ajout d'un nouveau client 
```sh
Looks like OpenVPN is already installed

What do you want to do?
   1) Add a cert for a new user
   2) Revoke existing user cert
   3) Remove OpenVPN
   4) Exit
Select an option [1-4]: 1
```
OpenVPN nous demande d'indiquer un nom pour le client. Pour ma part je veux connecter mon téléphone au VPN. Ce nom n'impactera rien du tout. (Autrement dit si vous avez un samsung et que vous rentrez Iphone ou MacBook, cela n'aura aucun effet negatif)
```sh
Tell me a name for the client cert
Please, use one word only, no special characters
Client name: Samsung_s10
```
OpenVPN donne comme resultat ceci :
```sh
Generating a 2048 bit RSA private key
.....+++
..................................+++
writing new private key to '...'
-----
Using configuration from /etc/openvpn/server/easy-rsa/openssl-1.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'Samsung_s10'
Certificate is to be certified until Jan 25 15:07:37 2027 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated

Client Samsung_s10 added, configuration is available at /Ubuntu/Samsung_s10.ovpn
```
OpenVPN a créé un fichier de configuration client (Samsung_s10.ovpn) utilisable sur Iphone, Mac, Android, Windows, Linux.
Il nous donne le chemin du fichier de configuration client "/Ubuntu/Samsung_s10.ovpn".

Il ne vous reste plus qu'a installer le client OpenVPN sur l'appareil désiré et de charger la configuration client (Samsung_s10.ovpn).
