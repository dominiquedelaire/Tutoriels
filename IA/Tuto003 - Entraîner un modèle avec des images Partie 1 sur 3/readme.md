# Introduction - Entraîner un modèle avec des images - Partie 1 sur 3
> Version 2025.09.18.1, Auteur : Dominique Delaire, Neurones Solutions Inc.
![Dermabots0](https://github.com/user-attachments/assets/654dd6e6-fd7f-4d7a-b02d-b7e6274e1788)


## Introduction
Dans un monde où les technologies de l'intelligence artificielle évoluent à une vitesse fulgurante, leur application dans le domaine médical devient de plus en plus cruciale. Parmi les nombreux défis que les professionnels de la santé doivent relever, la détection précoce du mélanome, une forme agressive de cancer de la peau, est particulièrement primordiale. C'est dans cette optique que cet article propose une exploration approfondie d'un système de machine learning capable de différencier les grains de beauté bénins des mélanomes potentiels à partir d'images dermatologiques.

Nous commencerons par une introduction à l'utilisation de Shellbots, notre environnement pour le développement de modèles de machine learning et d'IA générative intégrant les moteurs principaux du marché (Pytorch, Google Tensor Flow, Dataiku, etc.). 

Dans la partie 2, nous montrerons comment intégrer efficacement Dataiku pour la préparation et l'analyse des données d'image. Puis pour voir les différences d'outils, nous plongerons dans l'utilisation de Pytorch, une bibliothèque de deep learning, pour entraîner un modèle capable d'identifier les caractéristiques distinctives d'un mélanome.   

Dans la partie 3, enfin, nous démontrerons comment ce modèle peut être intégré dans Shellbots pour offrir un outil précieux aux médecins, dermatologues et patients.

Notre solution ne vise pas seulement à automatiser la détection, mais également à fournir un soutien décisionnel aux professionnels de santé, augmentant ainsi les chances de détection précoce et améliorant les résultats cliniques. 

La solution peut être adaptée à vos besoins cliniques.

Bien sûr à des fins de formation, nous allons simplifier un peu le tutoriel, mais vous allez pouvoir découvrir comment les technologies de l'IA peuvent transformer le domaine médical, offrant des solutions innovantes et accessibles pour lutter contre l'une des menaces les plus graves pour la santé humaine. 

L'IA ce n'est pas simplement de l'IA générative et cet article se veut une initiation pour entraîner un modèle simple ici dans nos exemples.

## Création d'un contexte dans Shellbots pour accueillir les données et images d'entraînement avec Dataiku

Dans la console du système d'exploitation Shellbots, nous **créons un contexte** avec la commande **"createcontext"**   

Un **contexte** est une **sorte de projet** Shellbots où on entrepose et **gère différentes choses** : nos **données, nos modèles, nos services** pour notre base de **machine learning et/ou d'IA générative**.   

Puis pour **changer de contexte et travailler sur celui-ci** dans Shellbots, il suffit d'utiliser la **commande "usecontext"** et de choisir son contexte spécifique.

![Dermabots1](https://github.com/user-attachments/assets/d6b27b9e-b621-4591-a268-e28d532b9c6a)


## Données de référence représentant des grains de beauté bénins ou malins

Pour simplifier le tutoriel, nous utiliserons ici des données de différentes sources :
* **ISIC (International Skin Imaging Collaboration)** : qui contient des centaines de milliers d'images de lésions cutanées y compris des mélanomes. Ces images sont annotées par des experts, ce qui en fait une ressource idéale pour l'entraînement de modèles d'apprentissage automatique.
* **PH2 Database** : Une autre base de données contenant des images de mélanomes et d'autres types de lésions cutanées acquise du service de Dermatologie de l'Hôpital de Matosinhos au Portugal.
* **D'autres sources anonymes de sources médicales**

Voici un **exemple de la base de données ISIC :**

![Capture d’écran, le 2024-08-31 à 17 40 13](https://github.com/user-attachments/assets/1875627b-24ff-405e-9457-df4852bab195)

et la **base données PH2** qui contient un outil de visualisation en matlab et où l'on peut télécharger les images et leur classification respective :
https://www.fc.up.pt/addi/ph2%20database.html

## Sélection des données

Pour l'exemple ici, nous avons sélectionné 3500 images présentant des grains de beauté avec un cas de mélanome et 3500 images présentant des grains de beauté bénin.

Nous avons essayé d'équilibrer nos données.

La prochaine étape va être de trier et de classifier les données dans un nouveau modèle. Pour cela, nous allons utiliser Dataiku intégré à Shellbots.

**Dans shellbots, la ligne de commande est aussi un prompt :)** Donc, si shellbots ne reconnait pas une commande, il va interpréter votre demande comme un prompt. La réponse à ce prompt dépend du moteur par défaut configuré dans le système d'exploitation Shellbots. Par défaut ici, c'est le fournisseur OpenAi avec le modèle Gpt-4o-mini mais vous pourriez le changer avec des fonctions dédiées. Shellbots contient la majorité des moteurs ia du marché et des modèles locaux spécialisés.

Avant de lancer Dataiku, voici les principales fonctions de Dataiku. 
**Pour cela, nous lui avons posé la question suivante : Qu'es ce que Dataiku ? :)**

![dermabots2](https://github.com/user-attachments/assets/a797183d-4bbe-4053-863f-e3dd3bafff5e)

Maintenant, pour lancer Dataiku, il suffit de taper la commande **dataiku**

![dermabots3](https://github.com/user-attachments/assets/d4abdfe1-6879-4fba-9e6c-81d041254401)

Et le navigateur par défaut va s'ouvrir pour afficher l'interface de connexion de Dataiku

![Capture d’écran, le 2024-08-31 à 21 13 45](https://github.com/user-attachments/assets/b57293a8-1c20-4467-887f-899c1280b46f)

**Dans le prochain article, nous traiterons de la préparation, l'analyse et la classification des données et des images avec Dataiku !**
