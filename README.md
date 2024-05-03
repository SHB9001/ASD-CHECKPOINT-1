# ASD-CHECKPOINT-1

## Partie 1 : Gestion des utilisateurs, groupes et droits

### 1.1 Manipulation : créer un compte utilisateur

```
sudo adduser --disabled-password --gecos "" wilder

sudo mkdir /home/share && sudo chown wilder:wilder /home/share/ && sudo chmod 700 /home/share

su wilder && touch /home/share/passwords.txt

sudo groupadd share

sudo usermod -aG share wilder

sudo usermod -aG share root

sudo chown -R :share /home/share && chmod -R g+rw /home/share && chmod -R g-x /home/share
```

Pour chiffrer le dossier :
```
sudo apt-get install gnupg tar 

tar -cf sharezipped.tar /home/share/
gpg --output sharezipped.tar.gpg --symmetric --cipher-algo AES256 sharezipped.tar
```

Pour dechiffrer le dossier :
```
gpg --output sharezipped.tar --decrypt sharezipped.tar.gpg
tar -xf sharezipped.tar
```

Pour le message de connexion :
```
echo -e "echo \"=============================\"" >> /home/wilder/.bashrc
echo -e "\n echo \" -- Bienvenue cher Wilder --\"" >> /home/wilder/.bashrc
echo -e "\n echo \"=============================\"" >> /home/wilder/.bashrc

host=$(hostname)
space=$(df -h / | awk 'NR==2 {print $4}')
ram=$(free -h | grep -i "mem" | awk '{print "Used:", $3}' | cut -d ' ' -f 2)

echo -e "\n echo \" - Hostname............: $host \"" >> /home/wilder/.bashrc
echo -e "\n echo \" - Disk Space..........: $space \"" >> /home/wilder/.bashrc
echo -e "\n echo \" - Memory used.........: $ram \"" >> /home/wilder/.bashrc

echo -e "\n echo \"=============================\"" >> /home/wilder/.bashrc
```

### 1.2 Question : quelles sont les adresses ip propre au conteneur ?

En fonction de la configuration.
Le conteneur aura accès aux autre conteneurs, soit il sera ouvert et récupèrera une IP (fixe ou DHCP).
Pour cet exercice, ma VM est en mode pont et a récupéré une IP locale avec ma box.

## Partie 2 : Sécurité du conteneur et durcissement ssh

### 2.1 Manipulation : sécuriser l'environnement distant

Il faut modifier le fichier /etc/ssh/sshd_config puis redémarrer le service (en l'occurence on passe au 3212).
```
sed -i '/^Port 22/s/.*/Port 3212/' /etc/ssh/sshd_config

sudo systemctl restart sshd
```

Avec ipTables :
```
# Flush iptables
iptables -F
iptables -X
ip6tables -F
ip6tables -X

# Passage de toutes les politiques IPv4 en DROP par defaut
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP

# Passage de toutes les politiques IPv6 en DROP par defaut
ip6tables -P INPUT DROP
ip6tables -P OUTPUT DROP
ip6tables -P FORWARD DROP

# Autorisation des boucles locales (IPv4, et IPv6 au cas ou)
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
ip6tables -A INPUT -i lo -j ACCEPT
ip6tables -A OUTPUT -o lo -j ACCEPT	

# Autorisation des connexions existantes
iptables -A INPUT -m conntrack --ctstate ESTABLISHED -j ACCEPT
iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED -j ACCEPT

# Autorisation SSH
iptables -A INPUT -p tcp --dport 3212 -m conntrack --ctstate NEW -j ACCEPT
```

### 2.2 Question : quel autre moyen simple peux-tu mettre en œuvre pour augmenter la sécurité du conteneur ?

L'éteindre quand il n'est pas utiliser.
Forcer la connexion SSH par clef et interdire de se connecter avec root.
Créer une jail avec un outil comme Fail2ban.
En restreindre l'accès avec un WAF en proxy.

## Partie 3 : Scripting Bash

### 3.1 Manipulation : script bash de connexion distante

Il faudra au préalable créer une paire de clefs et que le serveur ssh cible accepte cette méthode de connection.

Pour générer une clef, lancer cette commande :
```
ssh-keygen -t rsa -b 4096 -C "[mail]"
```

Copier la clef publique dans /home/wilder/.ssh/authorized_keys (créer le dossier s'il n'existe pas), ou utiliser la commande :
```
ssh-copy-id wilder@[ip du serveur]
```

Pour se connecter :
```
ssh wilder@[ip du serveur]
```

### 3.2 Question : Qu'est censé faire le script Bash ci-dessous ?

Si l'utilisation CPU dépasse strictement les 95%, ce script :
- envoie un mail à 'wilder@email.sh' contenant le texte 'Running out of CPU power',
- envoie un message dans le terminal 'Percent used: [pourcentage]', pourcentage étant le taux d'utilisation du CPU.

### 3.2 Question : Est-ce que l'utilisateur "wilder" va pouvoir installer des paquets logiciels tels que apache ou nginx ? Que la réponse soit oui ou non, expliquer pourquoi en quelques mots

Non, car l'utilisateur n'a pas les droits sur la commande 'apt install' et qu'il ne fait pas partie des sudoers.

## Partie 4 : 10 questions sur le thème DevOps

### 4.1 Qu'est-ce que l'infrastructure as code (IaC) ?

Il s'agit d'un ensemble d'outils permettant de gérer une infrastructure informatique via du code, souvent sous forme de .yaml .
Ces outils sont particulièrement utiles pour l'automatisation, la fiabilité (les scripts testés sont plus fiables que les humains), la répétabilité et la 'scalability'.

### 4.2 Est-ce que Docker est une nécessité dans le milieu DevOps ? (Expliquer la réponse)

Non, il est possible de faire sans, mais cet outila tendance à s'imposer comme une évidence dans le contexte DevOps, du fait de sa flexibilité.
Notament en lien avec l'IaC, Docker permet une gestion automatisée et fine des ressources.

### 4.3 Qu'est-ce qu'une pipeline CI/CD ?

Continuous Integration / Continuous Development.
Il s'agit d'automatiser la chaîne de production du code.
Du terminal du développeur aux environnements de production en passant par les phases de tests, le CI/CD (et son application avec la pipeline) vont tenter d'automatiser un maximum d'étapes afin de permettre des itérations courtes et gagner en efficacité.

### 4.4 Quel outil (logiciel) utiliserais tu pour gérer des configurations serveurs à distance ?

Ansible, Puppet, Terraform pour la création et l'installation.
SSH pour le dépannage et l'administration 'fine'.

### 4.5 Que signifie le terme "scalabilité" pour le milieu DevOps ?

Il s'agit de la capacité de l'infrastructure à accomper les variations de besoins en ressources (en général leur croissance).
Un bon exemple de ça est l'utilisation dynamique de ressources cloud comme AWS, le DevOps tentera autant que possible d'automatiser la création (et destruction vu les prix !) de ressources si la charge (CPU, Ram, disque...) atteintle plafond actuel.
A noter que cela peut aussi être dangereux avec des fournisseurs cloud, si l'automatisme s'emballe pour n'importe quelle raison, la facture risque d'être salée !

### 4.6 Quel est le principal rôle d'un administrateur système DevOps en entreprise ?

Le DevOps a pour rôle de créer, maintenir et faire évoluer la chaîne de production du code automatisé (CI/CD).

### 4.7 Quel outil (plateforme) utiliserais tu pour créer une pipeline de déploiement logiciel ?

Github et Gitlab ont leurs outils propre. Jenkins est aussi un outil connu en la matière.

### 4.8 Quels types d'environnements mettrais tu en place avant une mise en production de logiciel ?

En général on a une division en 3 environnement :
- SIT, un environnement 'bac à sable' pour les dév,
- UAT, une préprod, très proche de la prod pour que les clients puissent valider les développement et faire leur recette,
- PRD, la production avec les produits finis et validés.

Cependant du fait des coûts (matériels et humains), on se retrouve souvent avec seulement deux environnements, prod et preprod.

### 4.9 Qu'est-ce que signifie exactement la notion d'intégration continue (CI) ?

Il s'agit du rassemblement du travail des différents développeurs travaillant sur un projet.
Cela inclus généralement l'automatisation de la vérification du code, s'assurer que celui-ci répond à des critères de qualité et ne provoque pas de régressions.
Cette automatisation permet d'effectuer un ensemble de vérification qui seraient autrement lourdes à faire manuellement à chaque itérations.

### 4.10 Que signifie la notion de "provisionning" pour un administrateur système DevOps ?

Il s'agit de la mise à disposition de ressources (informatiques dans notre cas). 
Cette mise à dispo peut-être manuelle ou automatisée, mais en DevOps on priviligéra l'automatique.
En DevOps cela couvre la mise en place de serveurs, de réseaux, de service et de configuration.

## Partie 5 : Administration système et réseau

### 5.1 Manipulation : installer un outil de surveillance de l'activité réseau d'un conteneur

Avec Wireshark et l'outil tshark pour l'utiliser en ligne de commande :
```
sudo apt update && sudo apt upgrade

sudo apt install wireshark tshark
```

Lancer l'analyse sur eth0 avec :
```
tshark -i eth0
```

A ce moment tout le trafique sur l'interface demandée est capturé. 
En ajoutant des options comme '-Y "http"' on pourra spécifier le type de trafique que l'on veut intercepter (dans l'exemple ici, le trafique HTTP).