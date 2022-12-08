# Installer OPVN incluant un PiHole sur Ubuntu
## _Une connnexion chiffreé et une suppression de pub !_

OpenVPN est un logiciel libre permettant de créer un réseau privé virtuel VPN. 
Le Pi-hole est un serveur DNS qui protège vos appareils des contenus indésirables sans installer de logiciel côté client.
`
- Difficulté : ★☆☆☆☆
- Puissance matérielle nécessaire : ★☆☆☆☆
- Temps : 20-45 minutes

## Prérequis matériels

- Un ordinateur qui devra rester constamment allumé ou un serveur dédié
- Une connexion internet stable et puissance côté serveur
- Un accès au shell de votre machine 

Pour ma part, j'utilise un serveur dédié de chez OVH sous Ubuntu Server 20.04 LTS "Focal Fossa". Une puissance réseaux de 100 Mb/s en montant et en descendant (Je n'ai jamais eu de soucis de ralentissement.)

## Installation
### Installation d'OpenVPN

Tout d'abord, nous allons mettre à jour nos dépôts puis mettre à jour la machine.
```sh
sudo apt update && sudo apt upgrade
```
Création d'un dossier pour stocker tous les fichiers que nous avons besoin pour installation et qui nous serviront pour la suite (Personnellement, je l'ai créé dans mon répertoire d'utilisateur.).
```sh
mkdir OPVN_PiHole
```
Nous allons nous rendre de ce dossier.
```sh
cd OPVN_PiHole
```
Nous allons télécharger OpenVPN, le rendre exécutable puis lancer l'installation.
```sh
wget https://raw.githubusercontent.com/RenardFurtif/opvn-pihole/main/openvpn-install.sh
chmod 755 openvpn-install.sh
sudo ./openvpn-install.sh
```
En tapant les commandes ci-dessus, nous entrons dans la face d'installation. L'installateur d'OpenVPN vous réclamera quelles informations.

L'installateur nous demande quel protocole nous voulons utiliser pour le tunnel VPN.

Choisir l'option "1" donc le protocole UDP
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
Ensuite, l'installateur nous demande un port pour le tunnel VPN. (Par défaut : 1194)

Choisir le port "1194"
```sh
What port do you want OpenVPN listening to?
Port: 1194
```
Puis l'installateur nous demande de choisir un serveur DNS (Par défaut : Current system resolvers)

Choisir l'option "1" donc le serveur DNS que vous utilisez actuellement sur votre machine. Vous pouvez choisir une autre option si un de vos serveurs DNS préféré se trouve dans cette liste.
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
Enfin, nous devons créer un nom pour une configuration client OpenVPN.

Entrez le nom "PiHole", vous pouvez changer celui-ci à votre guise. 
```sh
Finally, tell me your name for the client certificate
Please, use one word only, no special characters
Client name: pihole
Okay, that was all I needed. We are ready to setup your OpenVPN server now
Press any key to continue...
```
Bravo ! L'installation OpenVPN est terminée.
```sh
Finished!

Your client configuration is available at /Ubuntu/pihole.ovpn
If you want to add more clients, you simply need to run this script again!
```

### Installation de PiHole
Toujours dans le dossier "OPVN_PiHole"

Nous allons télécharger PiHole et lancer l'installation. 
```sh
wget https://raw.githubusercontent.com/RenardFurtif/opvn-pihole/main/pihole-install.sh
sudo bash pihole-install.sh
```
Choisissez tun0 comme interface et 10.8.0.1/24 comme adresse IP. Vous pouvez accepter le reste des valeurs par défaut.

Bravo ! L'installation de PiHole est terminée.

## Configuration 
### Configuration d'OpenVPN

Tout d'abord, nous devons modifier le fichier config d'OPVN.
```sh
sudo nano /etc/openvpn/server/server.conf
```
Ensuite, changer cette ligne.
```sh
push "dhcp-option DNS 8.8.8.8"
```
Par cette ligne.
```sh
push "dhcp-option DNS 10.8.0.1"
```
Cette ligne indique aux clients se connectant au VPN qu'ils doivent utiliser Pi-hole comme serveur DNS primaire.

Maintenant, nous devons redémarrer le VPN. Si cette commande ne marche pas redémarrer entièrement la machine
```sh
systemctl restart openvpn-server@server
```
Bravo ! La configuration d'OpenVPN est terminée.
### Configuration du pare-feu de PiHole 

Cette étape est la plus importante, c'est elle qui va permet de bloquer les publicités.

Ces commandes autoriseront le DNS et le HTTP nécessaires à la résolution de noms de domaines (en utilisant Pi-hole comme serveur dns) et l'accès à l'interface Web.
```sh
iptables -A INPUT -i tun0 -p tcp --destination-port 53 -j ACCEPT
iptables -A INPUT -i tun0 -p udp --destination-port 53 -j ACCEPT
iptables -A INPUT -i tun0 -p tcp --destination-port 80 -j ACCEPT
```
Vous allez activer l'accès SSH et VPN depuis les ip publics et locales.
```sh
iptables -A INPUT -p tcp --destination-port 22 -j ACCEPT
iptables -A INPUT -p tcp --destination-port 1194 -j ACCEPT
iptables -A INPUT -p udp --destination-port 1194 -j ACCEPT
```
Le paramètre crucial suivant est d'autoriser TCP/IP à effectuer des "three-wayhandshakes" :

```sh
iptables -I INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
```
Nous allons autoriser tout trafic de bouclage, c'est-à-dire que le serveur est autorisé à se parler à lui-même sans aucune limitation en utilisant 127.0.0.0.
 
 ```sh
iptables -I INPUT -i lo -j ACCEPT
```

Puis, nous voulons rejeter l'accès provenant de n'importe où (c'est-à-dire si aucune règle ne correspond alors nous rejetons le trafic).
 ```sh
iptables -P INPUT DROP
```
Nous allons également en profiter pour bloquer les publicités HTTPS afin d'améliorer le blocage des publicités qui sont chargées via HTTPS et également traiter QUIC.
 ```sh
iptables -A INPUT -p udp --dport 80 -j REJECT --reject-with icmp-port-unreachable
iptables -A INPUT -p tcp --dport 443 -j REJECT --reject-with tcp-reset
iptables -A INPUT -p udp --dport 443 -j REJECT --reject-with icmp-port-unreachable
```
Point de vérification de l'iptables, pour vérifier si vous n'avez pas oublié de règles.
Nous allons afficher la liste des règles du pare-feu.

```sh
iptables -L --line-numbers
```
Pour mon cas, j'ai cette réponse. Si tout s'est bien passer, vous avez le résultat.
```sh
Chain INPUT (policy DROP)
num  target     prot opt source               destination
1    ACCEPT     all  --  anywhere             anywhere
2    ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
3    ACCEPT     udp  --  anywhere             anywhere             udp dpt:openvpn
4    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:domain
5    ACCEPT     udp  --  anywhere             anywhere             udp dpt:domain
6    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http
7    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
8    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:openvpn
9    ACCEPT     udp  --  anywhere             anywhere             udp dpt:openvpn
10   REJECT     udp  --  anywhere             anywhere             udp dpt:80 reject-with icmp-port-unreachable
11   REJECT     tcp  --  anywhere             anywhere             tcp dpt:https reject-with tcp-reset
12   REJECT     udp  --  anywhere             anywhere             udp dpt:443 reject-with icmp-port-unreachable

Chain FORWARD (policy ACCEPT)
num  target     prot opt source               destination
1    ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
2    ACCEPT     all  --  10.8.0.0/24          anywhere

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination
```
Si tout semble bien s'être bien passer, nous allons enregistrer nos règles afin de pouvoir y apporter des modifications plus tard.
```sh
iptables-save > /etc/pihole/rules.v4
```
Bravo ! La configuration du pare-feu de PiHole est terminée.

## Ajout d'un client au VPN
Toujours dans le dossier "OPVN_PiHole"
Maintenant, que tout fonctionne, il faut créer un nouvel utilisateur dans la configuration de notre VPN afin de pouvoir nous y connecter.

Nous allons lancer l'installateur OpenVPN qui est devenu une sorte de boîte à outils OpenVPN.
```sh
bash openvpn-install.sh
```
OpenVPN nous dit que le serveur est bien installé. Nous allons choisir l'option "1" donc l'ajout d'un nouveau client. 
```sh
Looks like OpenVPN is already installed

What do you want to do?
   1) Add a cert for a new user
   2) Revoke existing user cert
   3) Remove OpenVPN
   4) Exit
Select an option [1-4]: 1
```
OpenVPN nous demande d'indiquer un nom pour le client. Pour ma part, je veux connecter mon téléphone au VPN. Ce nom n'impactera rien du tout. (Autrement dit si vous avez un Samsung et que vous rentrez IPhone ou MacBook, cela n'aura aucun effet innattendue).
```sh
Tell me a name for the client cert
Please, use one word only, no special characters
Client name: Samsung_s10
```
OpenVPN donne ceci comme resultat :
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
OpenVPN a créé un fichier de configuration client (Samsung_s10.ovpn) utilisable sur IPhone, Mac, Android, Windows, Linux.
Il nous donne le chemin du fichier de configuration client "/Ubuntu/Samsung_s10.ovpn".

Il ne vous reste plus qu'à installer le client OpenVPN sur l'appareil désiré et de charger la configuration client (Samsung_s10.ovpn).

## Interface WEB de PiHole
### Connection à l'interface 

Cette interface va vous permettre le nombre total de requêtes, le nombre de requêtes bloquer, le pourcentage de requêtes bloquées, votre liste de sites blacklistés, quelques graphiques et de paramétrer pihole sans passer par un shell néanmoins toutes les options ne sont pas disponible sur cette interface donc pour de grosse modification rendez-vous sur le shell de votre machine.

Pour accéder à cette interface, il faut être connecté au VPN car celle-ci s'exécute en local.

Cette interface web se trouve à cette adresse depuis n'importe quel navigateur web.
```
http://pi.hole/
```
Une page de connexion s'offre à vous, je parie que vous ne connaissez pas le mot de passe :)

Pas de problème rendez-vous sur le shell de votre machine serveur.
Entrez cette comande.
```sh
sudo pihole -a -p
```
Entrez 2 fois votre nouveau mot de passe et le tour est joué !

### Optionnel : Activer HTTPS pour l'interface WEB PiHole

Cette étape est optionnelle, elle permet d'activer HTTPS à votre interface. Elle vous permettra de communiquer avec l'interface avec une couche de chiffrement afin d'éviter que des pirates récupèrent votre mot de passe PiHole en sniffant votre trafic réseaux. (si votre mot de passe PiHole est le même que votre mot de passe d'accès SSH de votre machine cela devient plus embêtant)

Création d'une clé de chiffrement et d'un certificat SSL
```sh
openssl req -newkey rsa:2048 -nodes -keyout pihole.key -x509 -days 365 -out pihole.crt
```
Ensuite, nous allons combiner les deux fichiers ensemble et supprimer les fichiers qui ne sont plus nécessaires.
```sh
cat pihole.key pihole.crt > combined.pem
sudo rm pihole.key pihole.crt
```
Nous allons déplacer le fichier "combined.pem" dans le répertoire approprié, un peu de rangement ne fait pas de mal.
```sh
sudo mv combined.pem /etc/lighttpd/ssl/combined.pem
```
Ensuite, nous nous rendons dans le dossier le lighttpd puis nous allons créer le fichier "external.conf" afin de configurer le serveur Web en mode https.
```sh
cd /etc/lighttpd/
sudo nano external.conf
```
Une fois que cela est fait copier coller le code juste en dessous dans "external.conf". 
```
+= (
  "mod_openssl"
)
# Ensure the Pi-hole Block Page knows that this is not a blocked d>setenv.add-environment = ("fqdn" => "true")

# Enable the SSL engine with a LE cert, only for this specific host$SERVER["socket"] == ":443" {
        ssl.engine = "enable"                                              ssl.pemfile = "/etc/lighttpd/ssl/combined.pem"
        ssl.honor-cipher-order = "enable"
        ssl.cipher-list = "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AE>        ssl.use-sslv2 = "disable"
        ssl.use-sslv3 = "disable"
}                                                                  
# Redirect HTTP to HTTPS                                           $HTTP["scheme"] == "http" {
        $HTTP["host"] =~ ".*" {
                url.redirect = (".*" => "https://%0$0")
        }
}
```
Voilà ! La configuration du HTTPS est terminée.

Il faut juste redémarrer le serveur lighttpd afin que les modifications soient prises en compte.
```sh
sudo service lighttpd restart
```
Il se peut que votre navigateur vous affiche un message de prévention, c'est normal. Le certificat n'a pas été émit par un tiers de confiance cependant le chiffrement est totalement fonctionnel. 
### Optionnel : Mise à jour automatique du PiHole

Cette étape est optionnelle, elle permet de mettre à jour PiHole, le service FTL et Lighttpd toutes les semaines à 5 h du matin tout les lundis. Les résultats des mises à jour seront stockés dans un fichier daté.

Tout d'abord, nous allons créer un fichier ou serons stockées les logs des mises à jour.
Toujours dans le fichier "opvn_pihole"
```sh
mkdir update_log
```
Ensuite, nous ouvrir le planificateur de tache. C'est l'outil crontab qui va nous permettre d'exécuter une commande à une date planifiée.
```sh
crontab -e
```
Crontab, nous retourne ceci :
```sh
# Edit this file to introduce tasks to be run by cron.
#
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
#
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').
#
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
#
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
#
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
#
# For more information see the manual pages of crontab(5) and cron(8)
#
# m h  dom mon dow   command
```
Enfin, nous allons créer une tache planifiée.

Elles permettent de lancer la commande "sudo pihole -up" tout les lundis. "> opvn_pihole/update_log/up-$(date +\%d_\%m_\%Y).log", cette partie permet d'enregistrer le résultat de la commande dans un fichier avec un nom daté à la date d'exécution de la tâche.

Ajouter cette ligne à la fin du fichier de configuration.
```sh
0 5 * * 1 sudo pihole -up > opvn_pihole/update_log/up-$(date +\%d_\%m_\%Y).log
```

Les logs des mises à jour seront disponibles dans le dossier "opvn_pihole/update_log/".
Les fichiers auront cette forme comme nom "up-21_11_2022.log". 

Bravo ! La création d'une tache planifiée afin de mettre à jour Pi Hole et de ces services est terminée.
