Générer un rapport de conformité avant et après remédiation
===
Introduction
---
Etapes:
1. Copie de clé et installation de l'agent sur la machine cliente
2. Préparation de la station
3. Scan de la machine cliente et analyse du rapport
4. Décision concernant les points à résoudre
5. Préparation de la remédiation et application de celle-ci
6. Validation de l'applications des remédiations
7. Plus loin

1. Copie de clé et installation de l'agent sur la machine cliente:
---
La première étape consiste en la préparation de la communication entre la station de contrôle et la machine cliente.
Pour commencer, il est nécessaire de générer des clés RSA afin de permettre d'avoir une communication chiffrée (SSH) et de s'authentifier par la même occasion.
Ensuite il est nécessaire de copier la clé publique sur la machine cliente et installer l'agent SCAP.

//Génération d'une paire de clés
ssh-keygen -b 4096 -t rsa -f $HOME/.ssh/id_rsa_srv2 -q -N ""
//Utilisation de la variable $target et copie de la clé publique sur la machine cible
target="srv2"
ssh-copy-id $target
//Installation de SCAP sur la machine  cible
ssh $target "yum -y install scap-security-guide"

2. Préparation de la station
---
La station ayant déjà été utilisée, il n'est pas nécessaire d'installer openscap-utils et scap-security-guide.
Il est toutefois nécessaire de télécharger les guides de sécurité qui sont différents des précédents.

//Accès aux guides et copie locale
curl -L https://git.io/JMPci \
-o /usr/share/xml/scap/ssg/content/ssg-rhel7-ds-1.2.xml
type="$target-bp028m-minimalbefore"
//Définition du data stream
data_stream="/usr/share/xml/scap/ssg/content/ssg-rhel7-ds-1.2.xml"
//Définition du dictionnaire
cpe_dict="/usr/share/xml/scap/ssg/content/ssg-rhel7-cpe-dictionary.xml"

3. Scan de la machine cliente et analyse du rapport
---


