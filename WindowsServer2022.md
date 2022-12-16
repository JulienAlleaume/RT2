### Initialisation du serveur :   

Pour commencer l'installation on a crée 4 disque virtuelle depuis le bios du serveur (3 disques de 500G vide et un dernier de 250G pour installer Windows Server) et activer la fonction SRIOV sur la carte réseau.

Installation de windows via la iDRAC en utilisant un ISO du site azure education et la clé fournis avec. On a utilisé la version windows server Datacenter 2022. La différence entre la version datacenter et la versions standard de windows server est (Je ne cite que les difference lié à hyper-v):    

|    Standard      |   Datacenter    |
| :--------------- |:--------------- |
| deux machines virtuelles plus un hôte Hyper-V par licence  |   nombre illimité de machines virtuelles, plus un hôte Hyper-V par licence      |
| Pas de rôle serveur  | Dispose des rôle serveur             |
| Oui (conteneurs Windows en nombre illimité ; jusqu’à deux conteneurs Hyper-V)  | Oui (conteneurs Windows et Hyper-V en nombre illimité)         |
|Pas de support hyper-v Host-Guardian | Supporte le hyper-v Host-Guardian |   

### Configuration de Windows :    

Après installation je crée le compte administrateur et me connecte (Ont a utilisé le même NOM et MDP pour les deux serveurs pour facilité la chose, mais on aurai pu crée un autre utilisateur sur les deux serveurs pour plus de securité)
J'arrive sur le bureau ou l'application `Gestionnaire de Serveur` s'affiche automatiquement à chaque démarrage.    

![ ](./SAEPhoto/bureau1.png)

Cette application permet de gérer les rôles du serveur et de le supervisé. Pour pouvoir me connecter à distance sans passer par l'iDRAC et avoir une meilleur experience visuel, j'active le paramètre `Bureau à distance`    

![ ](./SAEPhoto/bureaudistant.png)

Je fixe une adresse IP aux serveurs. Sous windows c'est dans Panneau de configuration -> Centre de réseau et partage -> Modifier les paramètres de la carte puis clique droit sur la carte et aller dans propriété. Sur Protocoles Internet versions 4 changer les propriétés et mettre la configuration de sont réseau.(IP, DNS, passerelle, etc...)    

Pour suivre, je m'y connecte grâce à `Remmina` sur linux, ou bureau distant sur windows.    

### Installations et réglages des rôles et fonctionnalités :    

Maintenant j'installe les rôles obligatoire au fonctionnement des serveurs Hyper-v pour pouvoir faire une migration à chaud(La chose la plus complexe à faire, car elle demande beaucoup de fonctionnalité supplémentaire à installer/configurer).    

Pour ajouter des rôles il faut aller sur `Gérer` en haut à droite du `Gestionnaire de serveur`.    

![ ](./SAEPhoto/ajoutrole.png)

Cliquer sur suivant dans l'onglets `avant de commencer`.   

![ ](./SAEPhoto/typeinstall.png)

Selectionner la première options, qui permet d'installer de nouveau rôles et fonctionnalité.   

![ ](./SAEPhoto/selectionsserveur.png)

Selectionner le serveur sur lequel vous voulez ajouter des rôles.   

![ ](./SAEPhoto/rolehyperv.png)

Le rôle hyper-v permet de crée un serveur qui gère la virtualisation et permet d'interconnecter plusieurs serveur hyper-v.   

![ ](./SAEPhoto/fonctioncluster.png)

Dans les fonctionnalité on selectionne 'clustering de basculement` qui permet de crée des clusters de serveur Windows.   

![ ](./SAEPhoto/confhyperv.png)

On arrive ensuite dans la configuration d'hyper-v.   

![ ](./SAEPhoto/commuhyperv.png)

Configuration d'hyper-v, je selectionne la carte réseau sur laquelle hyper-v vas crée sont commutateurs virtuel.    

![ ](./SAEPhoto/migrahyperv.png)

On peut activer et choisir le type de'authentification lors d'une migration. Etant donné que ce serveur vas être clusterisé on active aucune option ici.   

![ ](./SAEPhoto/emplacementhyperv.png)

On peut selectionner l'emplacement de stockage des disques durs virtuels et des fichiers de configuration des ordinateurs virtuels.   

Après ça j'installe `Windows Admin Center` qui permet d'acceder par web au informations du serveur et de l'administrer.    

![ ](./SAEPhoto/installwincenter.png)

Suivre les intruction et je conseille de désactiver les mises à jours microsoft update pour l'installation d'admin center afin d'éviter tout problème on pourra les effectuer après sans soucis.   

![ ](./SAEPhoto/updatemicro.png)

Activer WinRM sur https uniquement afin d'éviter des failles de sécurité et bug de droits plus tard.   

![ ](./SAEPhoto/centerwinrm.png)

Je décide de garder le port générique et de redirigé le trafic HTTP vers HTTPS pour éviter des conflits d'authentifications lors de la migration des VMs.   

![ ](./SAEPhoto/centerhttp.png)

Pour ce connecter au site il faut utilisé de base le nom de l'ordinateur (ex : `https://WIN-XXXXXXXX`) et s'identifier avec un utilisateur de la machine local.    
Une fois connecter voila l'interface que l'ont obtient :    

![ ](./SAEPhoto/tdbweb.png)


### Mise en place de la migration dynamique :   

Pour pouvoir faire la migration dynamique il faut crée un domaine et y mettre les deux serveurs.

Il faut ajouter le rôle service AD DS (active directory domain service). J'ai crée un domaine de manière rapide et sans pousser dans les détails.

Activation du rôle.

![ ](./SAEPhoto/servicedom.png)

Poursuite de l'installation.

![ ](./SAEPhoto/serviceADDS.png)

Création du nom de domaine, cocher la case `Ajouter une foret`, pour crée un nouveau domaine. Lui choisir un nom de type `nom.local`

![ ](./SAEPhoto/creadom.png)

Choisir la version et un mdp pour le gestionnaire de domaine. 

![ ](./SAEPhoto/active.png)

Ne rien toucher dans les options DNS, le message d'erreur est apapru car il detecte le DNS de l'iut.

![ ](./SAEPhoto/deleguation.png)

LE nom de domaine NetBios definis à partir de notre nom de domaine, il peut être changer.

![ ](./SAEPhoto/netbios.png)

Les différrent chemin d'accès je les laisse par défault.

![ ](./SAEPhoto/acces.png)

Finir l'installation et lancer le gestionnaire DNS pour finr la configuration.   
Une fenêtre récapitulative souvrira, cliquer sur suivant et terminer.   
Maintenant il faut mettre dans le deuxieme serveur uniquement l'ip du DNS de ce serveur de domaine et s'assurer qu'ils sont sur le même domaine.

Dans le pare feu activer la regles iSCSI dans les deux sens (entrer et sortie) pour permettre ici la migration de stockage des VM entre serveurs.

![ ](./SAEPhoto/parefeu.png)

### Connection des serveurs :   


Création du cluster de serveur.
Ajout des deux serveurs au cluster à l'aide du `Gestionnaire de cluster` afin qu'ils soit connecter ensemble. Pour permettre une meilleur fonctionnement des applications.   

Selection des serveurs qui vont êtres ajouter au cluster.

![ ](./SAEPhoto/instaclus1.png)

Je décide de ne pas executer le test car je vais le faire après l'installation.

![ ](./SAEPhoto/instaclus2.png)

Selection du nom du cluster.

![ ](./SAEPhoto/instaclus3.png)

Confirmation des décisions.

![ ](./SAEPhoto/instaclus4.png)

![ ](./SAEPhoto/instaclus5.png)

![ ](./SAEPhoto/instaclus6.png)

Voilà le cluster est crée et opérationnel mais toute les fonctionnalaité apporté par le cluster ne sont pas active. 

Maintenant relié les deux serveur hyper-v entre eux.    

Pour ce faire lancer le Gestionnaire Hyper-v. Ensuite dans la liste à guauche cliquer droit sur gestionnaire Hyper-v et selectionner `Se connecter au serveur`, mettre dans autre ordinateur le nom du serveur que l'ont souhaite ajouter (ex:`WIN-XXXXXXXX`).   


![ ](./SAEPhoto/hyperv2.png)

Voilà à quoi devrai ressembler les deux serveurs connecter.
Activer la migration dynamique sur les deux serveur :

![ ](./SAEPhoto/activemig.png)

Je crée une VM pour essayer la migration dynamique. J'utilise une Alpine.

Création VM rapide : 

![ ](./SAEPhoto/vm1.png)

![ ](./SAEPhoto/vm2.png)

![ ](./SAEPhoto/vm3.png)

![ ](./SAEPhoto/vm4.png)

![ ](./SAEPhoto/vm5.png)

Après ça je la démarre et commence la migration à chaud DE PLUS le lecteur virtuel ISO est toujours attaché ce qui n'est pas possible avec la concurrence :

![ ](./SAEPhoto/md1.png)

Cliquer droit sur la VM -> Déplacer

![ ](./SAEPhoto/md2.png)

Passer la première fenêtre

![ ](./SAEPhoto/md3.png)

Selectionner déplacer l'ordinateur virtuel pour pouvoir déplacer l'integralité de la VM.

![ ](./SAEPhoto/md4.png)

Je selectionne le serveur vers lequel migrer ma VM.

![ ](./SAEPhoto/md5.png)

Je choisis de tout migrer.

![ ](./SAEPhoto/md6.png)

Dossier de destination de la VM sur l'autre serveur.

![ ](./SAEPhoto/md7.png)

Commencement de la migration.

![ ](./SAEPhoto/md8.png)

Dans le statut de la VM ont peu voir l'avancement du transfert.

![ ](./SAEPhoto/md9.png)
![ ](./SAEPhoto/md10.png)

Arriver de la VM sur le serveur de destination sans problèmes.

![ ](./SAEPhoto/md11.png)

Comme vu la VM a migrer à chaud du serveur WIN-07... vers le WIN-5U... Et ce sans coupure durant le transfert. Mais il y à juste eu une 'actualisation' de la fenêtre de la VM mais je ne pense pas que ce genre de chose soit visible sur un ssh ou bureau distant. 

La migration à froid fonctionne aussi et dans la même configuration des choses.

Voici toute les technique tenter qui n'ont pas aboutis a un résultat positif : 


1: Création d'un disque virtuel iSCSI pour les VM :   

J'ai voulut crée un disque iSCSI pour pouvoir mettre le stockage des vm sur un disque partagé virtuel, mais j'ai eu différent problèmes que j'ai eu du mal à règler.

2: Création d'une pool de stockage dédier :   

J'ai essayer de crée une pool de stockage spécial pour les VM mais malheuresement les disque virtuel crée dans le bios de 500Go n'ont pas été reconnu par le cluster donc impossible d'aller plus loins.

3: Création d'un cluster complet pour faciliter la gestion d'Hyper-V:   

Comme dit précedement la partie stockage du cluster à été trés difficile voir impossible à faire, de plus les serveur étant sur un réseau non-professionel le cluster n'acceptait pas la configuration. Il etait possible de faire un test complet ou précis de sont cluster en voila une image du rapport de sortie avec des erreurs : 

![ ](./SAEPhoto/erreurclus.png)