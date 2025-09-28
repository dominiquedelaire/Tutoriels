
# Introduction à notre langage AIx pour l'IA
> Version 2025.09.27.1, Auteur : Dominique Delaire, Neurones Solutions Inc.



J'ai créé **AIx** en 2019 au départ pour accélérer le développement de services IA, notamment pour les systèmes d'exploitations Linux dont notre ShellbotsOS, OS dédié à l'IA. Il est intégré à notre framework Shellbots.ai.

**AIx** est une couche au dessus de python et qui permet rapidement de créer des chatsbots, des modèles, des apps, de gérer des données, de créer des bases de données vectorielles à la demande, de pouvoir les alimenter, interroger, etc...

J'ai créé un **transpiler** qui permet d'avoir une couche de base en python, langage de prédilection pour l'IA :)

**Exemple d'un code source AIx pour l'interrogation de données d'un logiciel de comptabilité :**
```aix
# HelloWorld.aix

# Charger une table SQL
dataset sales = sql "sqlite:///sales.db" table "orders"

# Créer une base vectorielle locale
vectorstore vs1 = chroma "db_chroma"

# Alimenter la base avec les données SQL
ingest sales into vs1

# Définir un chatbot local
chatbot bot1 = local "llama3:8b"

# Attacher la base vectorielle au chatbot
attach bot1 vs1

# Lancer une session de chat
chatwith bot1
```
