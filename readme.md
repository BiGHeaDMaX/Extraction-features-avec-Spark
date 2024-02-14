# **Réaliser un traitement dans un environnement Big Data sur le Cloud avec Spark**

## **Introduction**
Ce projet est basé sur le dataset [Fruits-360](https://www.kaggle.com/datasets/moltean/fruits), qui contient 90k photos détourées de fruits et légumes.

Nous allons procéder à deux traitements : 
- Extraction de features d'images à l'aide d'un modèle préentrainé (transfer learning).
- Réduction de la dimensionnalité des features extraites à l'aide d'une PCA et export des résultats dans un fichier CSV.

Ces traitements seront réalisés à l'aide de Spark, le but étant que ces derniers puissent facilement être mis à l'échelle.

## **Contenu de ce repository**
- **01 - Traitements (local).ipynb** : test de notre scipt en local pour vérifier son bon fonctionnement.
- **02 - Traitements (AWS-EMR).ipynb** : notebook a éxécuter sur le cloud avec un plus grand nombre d'images.
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
- Ouvrir log4j2.properties avec gvim par exemple (mode edition en appuyant sur "i")
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

Pour installer la nouvelle version d'AWS CLI (v2xx), la procédure se [trouve ici](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)<br>
On peut vérifier que c'est bien installé (et où) en tapant : which aws (sous linux)<br>
Pour installer la v1xx (non recommandé) : *pip install awscli*<br>

**Création d'une clef :**<br>
Aller dans son compte AWS, en haut à droite cliquer sur son nom puis sur *Informations d'identification de sécurité*.<br>
Sur cette page créer une clef d'accès. **Attention** : penser à la supprimer/désactiver quand elle ne sert plus.<br>
Une fois la clef crée, la télécharger au format csv.<br>

**Configuration d'AWS CLI :**<br>
Taper : aws configure<br>
Et entrer la clef précédemment crée (*Access key ID* et *Secret access key*).<br>
*Default region name* [None], pour mettre pour la France : eu-west-3<br>
*Default output format* [None], laisser par défaut en appuyant sur entrée.<br>
On peut aussi utiliser json comme *Default output format*, ce qui indique que vous souhaitez obtenir des réponses de l'API au format JSON.<br>

