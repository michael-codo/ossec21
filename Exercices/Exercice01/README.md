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
7. Conclusion

Copie de clé et installation de l'agent sur la machine cliente:
---
La première étape consiste en la préparation de la communication entre la station de contrôle et la machine cliente. Pour commencer, il est nécessaire de générer des clés RSA afin de permettre d'avoir une communication chiffrée (SSH) et de s'authentifier par la même occasion. Ensuite il est nécessaire de copier la clé publique sur la machine cliente et installer l'agent SCAP.  

code
//Génération d'une paire de clés  
ssh-keygen -b 4096 -t rsa -f $HOME/.ssh/id_rsa_srv2 -q -N ""  
//Utilisation de la variable $target et copie de la clé publique sur la machine cible  
target="srv2"  
ssh-copy-id $target  
//Installation de SCAP sur la machine  cible  
ssh $target "yum -y install scap-security-guide"  

Préparation de la station
---
La station ayant déjà été utilisée, il n'est pas nécessaire d'installer openscap-utils et scap-security-guide. Il est toutefois nécessaire de télécharger les guides de sécurité qui sont différents des précédents.

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

Le rapport "srv2-bp028minimal-before-report.html" affiche un total de 39 points testés dont 16 ont échoués pour un résultat de 87% étant donné qu'il s'agit là du niveau minimal, on peut penser qu'il faudrait obtenir 100% peu importe le contexte. Afin d'avoir un point de comparaison, le scan a été également fait en niveau intermédiaire. Dans ce cas, le résultat est nettement revu à la baisse, 151 points de contrôle dont 100 ont échoués pour un résultat de 45%.

Décision concernant les points à résoudre
---
La sécurité est un point essentiel en informatique malgré tout remédier à tous les points d'un rapport de sécurité peut, dans certains cas, être plus risqué.
Un serveur en production pourrait rencontrer des problèmes lors de l'application de remédiations, interrompant ainsi les services ce qui aurait bien évidemment un effet contre-productif. C'est pourquoi il est important de planifier et tester toutes ces routines de maintenance en dehors d'un environnement de production.
C'est dans ce cadre, qu'il a été décidé de remédier à 2 problèmes faisant partie de la même catégorie:
- xccdf_org.ssgproject.content_rule_accounts_maximum_age_login_defs
- xccdf_org.ssgproject.content_rule_accounts_password_minlen_login_defs

Préparation de la remédiation et application de celle-ci
---
//Scan d'une règle (répetée une 2ème fois pour rule2)  
rule1="xccdf_org.ssgproject.content_rule_accounts_maximum_age_login_defs"  
type="rule1-$target-bp028minimal-before"  
profile="xccdf_org.ssgproject.content_profile_anssi_nt28_minimal"  
oscap-ssh --sudo root@$target 22 xccdf eval \  
--fetch-remote-resource \  
--profile $profile \  
--results $type-results.xml \  
--report $type-report.html \  
--oval-results \  
--cpe $cpe_dict \  
--rule $rule1 \  
$data_stream  
//Génération du guide de configuration  
oscap xccdf generate guide \  
--profile $profile \  
--output $type-guide.html \  
$type-results.xml  
//Génération d'une remédiation Ansible (répetée une 2ème fois pour rule2)  
result_id=$(oscap info rule2-$type-results.xml | grep 'Result ID' | sed 's/[[:blank:]]Result ID: //')  
oscap xccdf generate fix \  
--fix-type ansible \  
--output rule1-$type-playbook.yml \  
--profile $profile \  
--result-id $result_id \  
rule1-$type-results.xml  

L'idée initiale était de générer les 2 remédiations mais de les déclencher à l'aide d'un seul fichier (via import_playbook):
- Set_Password_Expiration_Parameters.yml
- rule1-srv2-bp028minimal-before-playbook.yml
- rule2-srv2-bp028minimal-before-playbook.yml

Pour une raison encore non-identifiée, bien que les résultats d'Ansible étaient bons, les paramètres ne semblaient pas avoir été modifiés. De ce fait, rule1*.yml a été utilisé et une fois la configuration confirmée, le fichier "maître" a été utilisé à nouveau avec succès.

Validation de l'applications des remédiations
---
Sagissant ici d'une double remédiation, pour obtenir un résultat plus visuel, un scan global a été effectué et celui-ci confirme bien le passage de 16 à 14 points en échec pour un résultat de 90%.

//Validation  
type="$target-bp028minimal-after"  
profile="xccdf_org.ssgproject.content_profile_anssi_nt28_minimal"  
oscap-ssh --sudo root@$target 22 xccdf eval \  
--fetch-remote-resource \  
--profile $profile \  
--results $type-results.xml \  
--report $type-report.html \  
--oval-results \  
--cpe $cpe_dict \  
$data_stream  

Conclusion
---
Des serveurs en production sont, par définition, exposés à tous types de menaces. C'est pourquoi il est important d'avoir un moyen de vérifier leur état et ce de façon réfulière. Dans le cas de SCAP, une bonne pratique pourrait être de planifier un scan de façon "régulière", par exemple, 1 fois par mois en faisant bien attention de s'assurer que les guides soient mis à jours. Il peut être également intéressant de garder un historique, par exemple 12 rapports (1an), afin de pouvoir observer l'évolution en terme de sécurité. Aussi, dans le cas d'un paramètre mis en échec alors que précédemment il ne l'était pas pourrait être un signe important (alerte) qu'il s'agisse d'un acte malveillant ou non.  
Au délà des tâches planifiées, il serait bon également de garder en tête un planning de remédiations (non urgentes) afin de continuellement sécuriser les machines de manière progressive.  
Afin de rendre les scans et les remédiations plus flexibles, il serait sans doute intéressant d'avoir des guides de sécurités et fichiers de remédiations qui regrouperaient uniquement certains domaines. Comme par exemple un guide de sécurité et un fichier de remédiation axé uniquement sur les mots de passe. Et cela, dans le but de ne pas être obligé de lancer un scan complet lorsque ce n'est pas nécessaire (SCAP Workbench).
Enfin, il est capital de traiter les "failles" en fonction des risques et non pas seulement en fonction du nombre de points en échec.






