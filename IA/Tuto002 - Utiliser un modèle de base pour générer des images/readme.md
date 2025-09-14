
![introimg](https://github.com/user-attachments/assets/d692ca83-3b70-4ef1-bee9-b113d9f444c2)
> Version 2025.09.14.1, Auteur : Dominique Delaire


**Cet exemple est l'un des vingt services disponibles pour les restaurants, chefs, hôtels, traiteurs, etc., grâce à notre service d'IA AiFood, créé avec notre framework Shellbots.**
C'est la première version de notre module qui utilisait Stability et qui maintenant utilise un modèle local spécifique et propre à chaque chef en fonction de son style de cuisine.

Vous pouvez utiliser le modèle simple shellbots et le convertir facilement en python en vous créant une clé api stability.

**Dans le rendu que vous verrez dans la démonstration vidéo (à la fin de cet article), le mode terminal ou scientifique est utilisé.**
L'utilisateur peut accéder à son service via le web, via une interface graphique. Ici, le rendu sert à tester le modèle et le service avant son déploiement final.
![image1](https://github.com/user-attachments/assets/5e76f303-5cca-41c6-b573-8d53318e3dff)


## Introduction
Dans Shellbots, nous pouvons ajouter et créer des modèles d'IA à partir de plus de 70 moteurs d'IA disponibles sur le marché. AiFood utilise plusieurs types d'IA, dont l'IA générative, notamment pour la génération de recettes, d'images de décoration ou la présentation de plats. Lorsque nous créons un modèle d'IA pour un service spécifique, nous pouvons utiliser un modèle de données d'apprentissage automatique associé. Par exemple, disposer d'une base ou d'un dictionnaire de goûts et de saveurs, ou de l'ADN des plantes/légumes, par exemple.

## Exemples de modèles d'IA de base
**Premier exemple :**
* Ici, pour créer une des 20 fonctions d'AIFood, nous avons établi un modèle pour la première version et tests avec stability afin de générer des images simples de décorations servant de présélection pour le chef.
* L'avantage de concevoir un modèle d'IA Shellbot est de pouvoir intégrer des fonctions du framework Shellbots en combinant du code Python.
* Un modèle d'IA dans le contexte d'AiFood définit une ou plusieurs fonctions du service d'IA.
* Vous pouvez créer autant de modèles que vous le souhaitez et les associer à des modèles de ML pour accroître la pertinence du résultat incluant des modèles locaux.
* Exemple très simple de modèle d'IA de base avec Stability sans paramètres spécifiques :
![exemplemodelai](https://github.com/user-attachments/assets/5bc28b38-1d83-415a-a5d7-b43883affbb6)

**Deuxième exemple :**
* L'utilisateur peut discuter ou suggérer des idées concernant une recette ou une recherche de plat, et l'un des modèles peut suggérer des suggestions basées sur un modèle ml, avec une base sur l'ADN des légumes par exemple.
* Dans ces cas, le modèle utilisera également un moteur llm si nous le définissons dans le modèle ia.
* Exemple de cas :
![exemplemodelai2](https://github.com/user-attachments/assets/92f3e328-e54e-4d77-9e9a-7c34a3ec232c)

## Vidéo simple démo dans ShellbotsOS
https://www.youtube.com/watch?v=TRcAd0secBo
![videoservice1demo](https://github.com/user-attachments/assets/fe1cc211-8a35-4ce5-9d7d-6c04e2833767)


