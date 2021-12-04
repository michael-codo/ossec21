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

Copie de clé et installation de l'agent sur la machine cliente:
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

Préparation de la station
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

Scan de la machine cliente et analyse du rapport
---
Avant de lancer un scan, il est nécessaire de sélectionner et identifier le profil à utiliser. Une fois le scan terminé il est alors possible de consulter les résultats au format .html

//Affichage des différents profils disponibles
oscap info --fetch-remote-resources $data_stream
//Profil minimal retenu pour l'exercice
profile=xccdf_org.ssgproject.content_profile_anssi_nt28_minimal
Note: intermediate_profile=xccdf_org.ssgproject.content_profile_anssi_nt28_intermediary
//Commande lançant le scan sur $target
type="$target-bp028minimal-before"
profile="xccdf_org.ssgproject.content_profile_anssi_nt28_minimal"
oscap-ssh --sudo root@$target 22 xccdf eval \
--fetch-remote-resource \
--profile $profile \
--results $type-results.xml \
--report $type-report.html \
--oval-results \
--cpe $cpe_dict \
$data_stream
//Activation d'un service http basique afin de consulter les rapports
python3 -m http.server 8080
Note: étant donné que firewalld est inactif il était, ici, inutile d'ajouter une nouvelle règle pour accèder à la station de contrôle en http

Le rapport "srv2-bp028minimal-before-report.html" affiche un total de 39 points testés dont 16 ont échoués pour un résultat de 87% étant donné qu'il s'agit là du niveau minimal, on peut penser qu'il faudrait obtenir 100% peu importe le contexte. Afin d'avoir un point de comparaison, le scan a été également fait en niveau intermédiaire.
Dans ce cas, le résultat est nettement revu à la baisse, 151 points de contrôle dont 100 ont échoués pour un résultat de 45%.

Décision concernant les points à résoudre
---
