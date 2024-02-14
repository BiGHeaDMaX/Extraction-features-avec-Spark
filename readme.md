# **Réaliser un traitement dans un environnement Big Data sur le Cloud avec Spark**

## **Introduction**
Ce projet est basé sur le dataset [Fruits-360](https://www.kaggle.com/datasets/moltean/fruits), qui contient 90k photos détourées de fruits et légumes.

Nous allons procéder à deux traitements : 
- Extraction de features d'images à l'aide d'un modèle préentrainé (transfer learning).
- Réduction de la dimensionnalité des features extraites à l'aide d'une PCA et export des résultats dans un fichier CSV.

Ces traitements seront réalisés à l'aide de Spark, le but étant que ces derniers puissent facilement être mis à l'échelle.

## **Contenu de ce repository**
- **01 - Traitements (local).ipynb** : test de notre script en local pour vérifier son bon fonctionnement.
- **02 - Traitements (AWS-EMR).ipynb** : notebook a exécuter sur le cloud avec un plus grand nombre d'images.
- **03 - Présentation.pptx** : slides de présentation du projet.

## **Mise en place de l'environnement Big Data**

### **1. Installation de Spark en local pour les tests**

Sous Windows, il faudra passer par WSL (que j'ai utilisé ici avec Ubuntu). Avant d'installer Spark, il faudra au préalable installer Java.<br>
Avant d'installer java, faire la modification suivante : ajouter /mnt dans PRUNEPATHS dans le fichier /etc/updatedb.conf<br>
Sinon l'installation de Java va prendre trop longtemps car une indexation sera lancée sur tous les fichiers windows ([plus d'infos ici](https://askubuntu.com/questions/1251484/why-does-it-take-so-much-time-to-initialize-mlocate-database)).<br>
<br>
Une fois cette modification faite, installer Spark. Un tutoriel est [disponible ici](https://computingforgeeks.com/how-to-install-apache-spark-on-ubuntu-debian/).<br>
<br>
Spark peut être très verbeux, pour réduire la verbosité :<br>
- Aller dans /opt/spark/conf
- Copier (commande cp) log4j2.properties.template pour créer log4j2.properties
- Ouvrir log4j2.properties avec gvim par exemple (mode édition en appuyant sur *i*)
- Chercher la ligne rootLogger.level = info et remplacer *info* par *error*
<br>
Lancer un script, exemple : <br>
/opt/spark/bin/spark-submit wordcount.py text.txt

### **2. Spark Shell**
Pour lancer la version python : *pyspark* ou */opt/spark/bin/pyspark*<br>
<br>
Pour un interpréteur un peu plus convivial : <br>
- Utiliser ipython (si pas disponible : pip install)
- Le mettre dans la variable PYSPARK_PYTHON
- Lancement avec la commande : *PYSPARK_PYTHON=ipython /opt/spark/bin/pyspark* ou *PYSPARK_PYTHON=ipython pyspark*
<br>
Dans un Spark Shell vous n'avez pas besoin de créer un objetSparkContext puisqu'il a déjà été créé. Il est disponible via la variable sc.

### **3. Installation d'AWS CLI**

Pour installer la nouvelle version d'AWS CLI (v2xx), la procédure se [trouve ici](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).<br>
On peut vérifier que c'est bien installé (et où) en tapant : which aws (sous linux).<br>
Une fois installé, il peut être nécessaire de relancer le terminal avant de pouvoir lancer une commande aws.<br>
Pour installer la v1xx (non recommandé) : *pip install awscli*<br>

**Création d'une clef :**<br>
Aller dans son compte AWS, en haut à droite cliquer sur son nom puis sur *Informations d'identification de sécurité*.<br>
Sur cette page créer une clef d'accès. **Attention** : penser à la supprimer/désactiver quand elle ne sert plus.<br>
Une fois la clef créée, la télécharger au format csv.<br>

**Configuration d'AWS CLI :**<br>
Taper : aws configure<br>
Et entrer la clef précédemment crée (*Access key ID* et *Secret access key*).<br>
*Default region name* [None], pour mettre pour la France : eu-west-3<br>
*Default output format* [None], laisser par défaut en appuyant sur entrée.<br>
On peut aussi utiliser json comme *Default output format*, ce qui indique que vous souhaitez obtenir des réponses de l'API au format JSON.<br>

### **4. Création d'un bucket sur S3**
Nous stockerons tous nos fichiers dans un bucket S3 (logs de Spark, notebook, images à traiter et données produites par les traitements).<br><br>
Pour créer le bucket : aws s3 mb s3://*Nom_Du_Bucket*<br><br>
Le nom doit être unique, pas seulement sur votre compte mais parmi tous les buckets créés par tous les utilisateurs sur S3. Ce qui est logique car ces buckets peuvent être accessibles publiquement (privés par défaut).<br><br>

**Upload de fichiers :**<br>
Pour uploader tout un dossier, par exemple celui contenant toutes les images, se déplacer dans le dossier et taper : aws s3 sync . s3://*Nom_Du_Bucket*/*Nom_Du_Dossier_De_Destination*<br><br>

Si vous n'êtes pas dans le dossier à uploader, indiquez le chemin complet de ce dernier à la place du "." après *sync*.<br><br>

**Autoriser l'accès public de certains fichiers :**<br>
Si vous souhaitez ouvrir au public l'accès à certains fichiers, sur le site d'AWS, aller dans le bucket créé, puis dans l'onglet *Autorisation*, puis désactiver *Bloquer l'accès public*. Par défaut l'accès publique de tout le contenu du bucket est désactivé, prudence quand vous désactivez ce paramètre car cela veut dire que vous pourrez alors ouvrir l'accès à certains fichiers.<br><br>
Toujours dans *Autorisation*, allez dans *Stratégie de compartiment* et vous pourrez y paramétrer l'accès à certains fichiers/dossiers, au format JSON. Exemple pour partager le notebook et le fichier de sortie de notre programme :<br><br>
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::Nom_Du_Bucket/jupyter/jovyan/02 - Traitements (AWS-EMR).ipynb"
        },
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::Nom_Du_Bucket/Results/pcaFeatures.csv"
        }
    ]
}
```

### **5. Création du cluster avec AWS EMR**
Sur le site d'AWS, aller sur EMR, puis : <br>
- Créer un nouveau cluster
- Donner un nom
- Choisir la dernière version d'Amazon EMR
- Cocher *Hadoop*, *Spark*, *Tensorflow* (si besoin de *keras*, il faudra l'installer en plus après avec un bootstrap) et *JupyterHub*. Laisser les autres qui sont cochés par défaut.

**Configuration de cluster :**<br>
- Laisser le choix par défaut *Groupe d'instances uniformes*.
- Pour ce projet, j'ai choisi des instances (primaire et unités) de type *m5a.xlarge* (les *m5.xlarge* ne semblaient pas marcher sur la région Paris au moment de la réalisation de ce projet).
- Retirer le groupe d'instance de tâches, pas nécessaire ici.
- Laisser le choix par défaut *Définir manuellement la taille du cluster*. Pour ce projet, j'ai choisi 2 Unités principales (workers). Donc on aura donc 3 instances EC2 : 1 instance maître (driver) et 2 instances principales (workers).

**Réseaux :**<br>
Cloud privé virtuel (VPC) et Sous-réseau : choisir ceux dans lequel se trouve votre bucket sur S3.<br>

**Résiliation du cluster :**<br>
Attention, dès que le cluster sera créé, vous allez commencer à payer, même lorsqu'aucun code n'est lancé.<br>
Il est recommandé de choisir *Résilier automatiquement le cluster après le temps d'inactivité* et de choisir une période au bout de laquelle le cluster sera *résilié*, c'est-à-dire supprimé.<br>

**Actions d'amorçage (bootstrap) :**<br>
Actions qui seront lancées au moment de la création de chaque instance. Nous allons créer un fichier shell .sh qui contiendra nos instructions, exemple des dépendances qui ont été installées pour ce projet : <br>
```
#!/bin/bash
sudo python3 -m pip install -U setuptools
sudo python3 -m pip install -U pip
sudo python3 -m pip install wheel
sudo python3 -m pip install pillow
sudo python3 -m pip install pandas==1.2.5
sudo python3 -m pip install pyarrow
sudo python3 -m pip install boto3
sudo python3 -m pip install s3fs
sudo python3 -m pip install fsspec
sudo python3 -m pip install keras
```

Remarque : j'ai ici installé une version un peu antérieure de *Pandas* (1.2.5), car il y avait un problème de compatibilité avec la version de *Numpy* installée par défaut sur les instances.<br>

Stocker ensuite ce fichier .sh sur votre bucket, puis cliquer sur *Ajouter* et indiquer le chemin vers le fichier .sh sur le bucket S3.<br>

**Journaux de clusters :**<br>
Choisir le chemin sur votre bucket S3 où vous souhaitez stocker vos logs.<br>
En procédant ainsi, vous pourrez encore consulter l'historique de l'exécution de votre code Spark après la résiliation du cluster.<br>

**Paramètres logiciels :**<br>
On va paramétrer la persistance des notebooks créés et ouvert via JupyterHub.<br>
Sans paramétrer ceci, les notebooks Jupyterhub seraient stockés par défaut sur le disque de l'instance EC2 maître et seraient donc perdus au moment de la résiliation du cluster.<br>
Ceci se paramètre au format JSON : <br>
```json
[
  {
    "Classification": "jupyter-s3-conf",
    "Properties": {
      "s3.persistence.bucket": "Nom_Du_Bucket",
      "s3.persistence.enabled": "true"
    }
  }
]
```

**Configuration de sécurité et paire de clés EC2 :**<br>
Choisir une paire de clé Amazon EC2 (qu'il faudra préalablement créer dans EC2/Réseau et sécurité/Paires de clés puis *Créer une paire de clef*, pensez à télécharger le fichier au format .pem et le mettre en lieu sûr).<br>

**Rôle Identity and Access Management (IAM) :**<br>
Si des rôles avaient été créés précédemment (précédentes créations d'EMR), les choisir, sinon en créer.<br>

**Création du cluster :**<br>
Cliquez enfin sur créer un cluster. L'opération peut prendre 15/20 minutes.<br>
Une fois le cluster créé et disponible, il sera affiché : *en attente*.<br>

**Recréer un nouveau cluster :**<br>
Même une fois résilié, votre cluster sera toujours visible dans votre liste de clusters sur EMR.<br>
Si vous souhaitez recréer un nouveau cluster identique, cliquez sur celui de votre choix, puis cliquez sur *Cloner* en haut à droite.<br>
Vous pouvez également cliquer sur *Cloner dans AWS CLI* pour faire la même opération dans AWS CLI.<br>

### **6. Connexion du master (driver) avec SSH pour accéder à Jupyterhub**
Il va falloir se connecter en SSH au driver (master) pour utiliser Jupyterhub.<br>
Mais le driver a par défaut un firewall qui bloque le port SSH (port 22).<br>
Il faut donc aller dans EC2, groupe de sécurité, chercher *sg-XXXXXXXXXXXXX - ElasticMapReduce-master* (le début du nom est variable).<br>
Modifier les règles pour ouvrir le port 22 (SSH), avec source = *N'importe où*. Faire ceci pour IPv4 et IPv6.<br>

**Connexion au driver en SSH :**<br>
Dans l'onglet propriété du cluster, chercher *DNS public du nœud primaire*.<br>
Puis, entrer la commande dans votre terminal : <br>
ssh -i Nom_Clef_EC2.pem hadoop@DNS_public_du_nœud_primaire<br>
Nom_Clef_EC2.pem : il s'agit de la clef que vous avez créé dans EC2.<br>
Bien mettre *hadoop@* avant l'adresse de votre cluster.<br>
La commande ci-dessus implique que vous vous trouviez dans le répertoire où se trouve la clef, sinon indiquer le chemin complet.<br><br>
Le fichier .pem doit avoir des droits d'accès restreints, sinon la procédure ne va pas fonctionner.<br>
**Sur Linux :** chmod 600 Nom_Fichier.pem<br>
Idéalement, placer le fichier.pem dans le dossier .ssh de votre dossier home.<br>
**Sur Windows :** c'est un peu plus fastidieux. Faire clic droit sur le fichier.pem, sécurité, avancé, supprimer les droits hérités, puis ajouter les droits à un seul utilisateur (par ex celui de la session utilisée) et ajouter seulement droits en lecture et écriture (pas exécution), [détails de la procédure ici](https://www.youtube.com/watch?v=AiWMy87o320).<br><br>
Remarque : quand vous créez uniquement une instance EC2 (et non un cluster EMR), pour vous connecter à cette dernière, il faudra mettre *ec2-user@Adresse_De_L'instance* et non *@hadoop".<br>

**Création d'un tunnel SSH :**<br>
Entrer dans votre terminal : ssh -i Nom_Clef_EC2.pem -D 5555 hadoop@DNS_public_du_nœud_primaire<br>
Remarque : j'ai choisi ici le port 5555, mais on aurait pu en choisir un autre.<br><br>
Ensuite, dans l'onglet Application du cluster on voit les liens des appli, dont JupyterHub.<br>
Pour y accéder, il faut utiliser un proxy sur le navigateur, j'ai choisi ici le plugin *foxyproxy*.<br>
Créer un accès dans foxyproxy : <br>
- Type : SOCKS5
- Hostname : localhost
- Post : 5555 (ou autre si on en a choisi un autre).
<br>
Ensuite on peut accéder à Jupyterhub, se connecter avec les identifiants par défauts : <br>

- login : jovyan<br>
- password : jupyter<br>
Uploader son notebook, puis cliquer dessus.<br>
Il faut bien choisir le kernel *pyspark* pour exécuter le notebook.

