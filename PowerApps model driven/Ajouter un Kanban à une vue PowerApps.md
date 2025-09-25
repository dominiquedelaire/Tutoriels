# Tutoriel Ajouter un composant Kanban dans une vue PowerApps
> Version 2025.09.21, Auteur : Dominique Delaire
>
![intro](https://github.com/user-attachments/assets/58b3eb76-faa0-478d-b045-1010f3717916))
![Capture d’écran, le 2023-10-07 à 14 29 07](https://github.com/nuage365/Tutoriels/assets/102873102/18dd8668-0cb7-4c55-b958-b5224eb72815)




Dans cet exemple, nous allons créer
 une nouvelle table Dataverse qui servira d'activités et de tâches à une table Dataverse nommée "Entités" qui gère des contacts, entreprises, fournisseurs, ...

La table des activités servira pour suivre des tâches, évènements, activités, rdv en lien avec une "Entité".

Le module permettra de gérer ses activités sous forme de Kanban !

# Prérequis
* Avoir un compte Microsoft PowerApps et un environnement de développement pour créer une solution
* Créer une solution dans votre environnement (dans notre exemple Mylife365 mais vous pouvez mettre n'importe quel nom)
* Les vues Kanban peuvent se faire uniquement sur des tables de type "Activités" ou "Opportunités".

# Partie 1 : Eléments de base
#### Etape 1 : Dans notre solution "Mylife365" (créez la votre), nous allons ajouter une nouvelle table.

![Capture d’écran, le 2023-10-07 à 13 55 00](https://github.com/nuage365/Tutoriels/assets/102873102/37b3510e-7d23-4924-9744-5aeebc5c1808)

#### Etape 2 : Nommez la "Activités et tâches" et choisissez comme type "Activités".

![Capture d’écran, le 2023-10-07 à 13 56 14](https://github.com/nuage365/Tutoriels/assets/102873102/687a4326-3d12-40ec-a219-ceac3416e303)

#### Etape 3 : Vous pouvez cocher aussi "Apparaître dans les résultats de recherche", puis enregistrez. Laissez les champs créés automatiquement tel quel pour la démo.

![Capture d’écran, le 2023-10-07 à 13 57 12](https://github.com/nuage365/Tutoriels/assets/102873102/cdf45e2c-a069-485e-a8ae-054f306cae42)

#### Etape 4 : Pour faire le lien entre cette nouvelle table et notre table existante de gestion de contacts et d'entreprise, il faut dans la table de gestion des contacts et d'entreprise, cocher la case "Faire de cette table une option quand : création d'une activité" 

![Capture d’écran, le 2023-10-07 à 14 00 42](https://github.com/nuage365/Tutoriels/assets/102873102/b1eec317-337b-4754-9486-bb050b24fac2)

#### Etape 5 : Notre table "Activités et Tâches" est bien présente dans la liste des objets de notre solution.

![Capture d’écran, le 2023-10-07 à 14 01 18](https://github.com/nuage365/Tutoriels/assets/102873102/652843be-f46b-4b6c-a73f-cc87cc2e99bc)

#### Etape 6 : On va maintenant ajouter notre entité dans le menu de l'application.

![Capture d’écran, le 2023-10-07 à 14 02 32](https://github.com/nuage365/Tutoriels/assets/102873102/2f311a47-e315-4e10-9589-f4f1d03f1ce3)

#### Etape 7 : Il suffit de sélectionner notre table dataverse créée

![Capture d’écran, le 2023-10-07 à 14 03 45](https://github.com/nuage365/Tutoriels/assets/102873102/5ade3530-2d85-4111-a90f-c2cb20aeb5ec)

#### Etape 8 : pour avoir notre nouvelle option dans le menu. Enregistrer et publier vos personnalisations.

![Capture d’écran, le 2023-10-07 à 14 03 58](https://github.com/nuage365/Tutoriels/assets/102873102/2a4f3236-c0ce-445e-9858-da0028a8801e)

#### Etape 9 : Ensuite, sélectionner votre application model driven dans la solution et sur le bouton droit de la souris. Sélectionner "Lire" pour lancer votre apps.

![Capture d’écran, le 2023-10-07 à 14 05 47](https://github.com/nuage365/Tutoriels/assets/102873102/ec177d77-e5f0-48fb-a789-c49b044ab287)

#### Etape 10 : On va tester maintenant et commencer à saisir des exemples dans notre environnement de développement pour notre nouvelle table.

![Capture d’écran, le 2023-10-07 à 14 06 20](https://github.com/nuage365/Tutoriels/assets/102873102/e092b3fa-df97-4dda-85fe-171853b61fb3)

#### Etape 11 : On choisit Activités et tâches, puis dans le bandeau du haut à droite, on clique sur les ... et on sélectionne "Activités et Tâches" pour créer une nouvelle activité.

![Capture d’écran, le 2023-10-07 à 14 06 52](https://github.com/nuage365/Tutoriels/assets/102873102/a9e45099-e059-471b-8f9a-7984c9897c9d)

#### Etape 12 : Saisir un Sujet et un concernant. Dans le concernant, il va pointer vers la table "Entité" contenant les contacts ou entreprises.
C'est automatique car il y a 2 tables dans notre solution PowerApps et grâce à l'étape 4 (cochez création d'une activité)

![Capture d’écran, le 2023-10-07 à 14 07 27](https://github.com/nuage365/Tutoriels/assets/102873102/c7664e51-2e00-493e-ae39-f5c0c5b72539)

#### Etape 13 : Sélectionner un enregistrement de la table "Entité" et saisir une date d'échéance par exemple avec une heure spécifique. Puis Enregistrez et fermer.

![Capture d’écran, le 2023-10-07 à 14 09 12](https://github.com/nuage365/Tutoriels/assets/102873102/2383d5a6-6905-46fd-813b-a28552fb0baa)

#### Etape 14 : Créez un deuxième enregistrement (Etape 11) mais cette fois ci dans le "Concernant", nous allons créer un nouvel enregistrement. Pour cela, cliquez sur "Créer enregistrement", puis choisir "Entités".

![Capture d’écran, le 2023-10-07 à 14 10 10](https://github.com/nuage365/Tutoriels/assets/102873102/9179e5ba-3557-4be1-aa4b-6fc9ca2c0da7)

![Capture d’écran, le 2023-10-07 à 14 10 21](https://github.com/nuage365/Tutoriels/assets/102873102/f76cf70a-69ec-45d0-a19d-29b1292dcfbb)

#### Etape 15 : Saisissez des données dans l'entité (ca peut être votre entité à vous). Puis cliquez sur "Enregistrer et fermer"

![Capture d’écran, le 2023-10-07 à 14 10 47](https://github.com/nuage365/Tutoriels/assets/102873102/875b7ec6-2224-4559-a58a-e23bef0f8f7d)

#### Etape 16 : Le "Concernant" a maintenant votre nouvelle donnée.

![Capture d’écran, le 2023-10-07 à 14 11 13](https://github.com/nuage365/Tutoriels/assets/102873102/d7b27ffe-0137-4d28-a27f-be11561975ca)

#### Etape 17 : Nous avons maintenant 2 nouvelles activités et tâches. Nous voyons par défaut que la vue présente les informations sous forme de liste.
L'objectif est de les présenter sous forme de KanBan.

![Capture d’écran, le 2023-10-07 à 14 12 48](https://github.com/nuage365/Tutoriels/assets/102873102/c10b382c-87ca-4cd3-9b56-ab4df0ed1630)

# Partie 2 : Définition du Kanban pour une vue spécifique
#### Etape 18 : Pour ajouter une vue Kanban sur l'ensemble des vues, il va falloir passer par l'ancienne interface. Pour cela, cliquez sur l'engrenage et choisir "Paramètres avancés".
![Capture d’écran, le 2023-10-07 à 14 12 48](https://github.com/nuage365/Tutoriels/assets/102873102/0e7b98dc-3267-48d5-8486-45686dc583f5)

#### Etape 19 : Puis dans le bandeau supérieur, choisir "Paramètres" puis l'option "Solutions". Sélectionner votre solution (la mienne ici est Mylife365)

![Capture d’écran, le 2023-10-07 à 14 14 32](https://github.com/nuage365/Tutoriels/assets/102873102/a5c31d11-8b19-4b4b-9dcf-c1003cf22e28)

#### Etape 20 : Double cliquer sur "Activités et Tâches" dans les entités de la solution. Puis choisir l'onglet "Contrôles". Choisir ensuite l'option "Ajouter un contrôle".

![Capture d’écran, le 2023-10-07 à 14 15 13](https://github.com/nuage365/Tutoriels/assets/102873102/5b62b9f8-e347-4be6-97f6-83fb129b7395)

#### Etape 21 : Sélectionner ensuite dans la liste le contrôle standard "Kanban" puis cliquer sur le bouton "Ajouter"

![Capture d’écran, le 2023-10-07 à 14 15 46](https://github.com/nuage365/Tutoriels/assets/102873102/0b4c573c-4962-430e-a7dd-9f957109dccb)

#### Etape 22 : Nous allons activer cette interface seulement pour web et tablette. Sur le téléphone, cela serait moins ergonomique surtout pour déplacer des enregistrements de case en case :) 
Si vous activez ce contrôle sur l'entité, toutes les vues auront ce contrôle par défaut. Si vous souhaitez qu'une vue spécifique est une vue Kanban est pas les autres, allez à l'étape suivante 23.

![Capture d’écran, le 2023-10-07 à 14 17 06](https://github.com/nuage365/Tutoriels/assets/102873102/f7c1654e-7d8e-4144-8ba5-070f12801599)

#### Etape 23 : On va faire la même chose mais au lieu de l'entité, on va choisir une vue spécifique existante pour appliquer la vue Kanban :)

![Capture d’écran, le 2023-10-07 à 14 17 59](https://github.com/nuage365/Tutoriels/assets/102873102/c3a9cf40-ed6b-4e6b-9ed9-a7444a095762)

#### Etape 24 : On va configurer la vue par défaut "toutes les activités et tâches". Pour cela, double cliquer sur la vue pour obtenir l'écran suivant.
Cliquez ensuite dans la section "Tâches courantes" sur l'option "Contrôles personnalisés".

![Capture d’écran, le 2023-10-07 à 14 19 00](https://github.com/nuage365/Tutoriels/assets/102873102/b0043509-a8f3-4e4e-9fa8-9d0e089b8e14)

#### Etape 25 : De la même façon que pour la table "Activités et tâches", ajouter le contrôle Kanban (voir étape 21)
Comme pour la table, on va juste avoir une vue Kanban en web et tablette. On n'active pas ce composant sur le mobile / téléphone.

![Capture d’écran, le 2023-10-07 à 14 19 12](https://github.com/nuage365/Tutoriels/assets/102873102/88110ec2-70d4-4395-8c95-874c09ff35a7)

![Capture d’écran, le 2023-10-07 à 14 19 26](https://github.com/nuage365/Tutoriels/assets/102873102/1dae120f-aee3-4b67-b9b5-0c483de32d1d)

![Capture d’écran, le 2023-10-07 à 14 19 45](https://github.com/nuage365/Tutoriels/assets/102873102/baf565b7-db20-4f6e-aa42-de70c25282bd)

#### Etape 26 : Une fois fais, on clique sur "Enregistrer et Fermer"

![Capture d’écran, le 2023-10-07 à 14 27 36](https://github.com/nuage365/Tutoriels/assets/102873102/1fcf1abe-06fa-4dc7-ac4b-27ac1375b9a0)

#### Etape 27 : Ne pas oublier de Publier toutes les personnalisations :)
![Capture d’écran, le 2023-10-07 à 14 27 50](https://github.com/nuage365/Tutoriels/assets/102873102/b5487914-7fa7-40d3-9fbb-be1583ef04ab)

#### Etape 28 : Maintenant on retourne dans notre environnement de développement dans l'application. On rafraichit la page en actualisant ou en faisant F5. Maintenant lorsqu'on clique sur le menu de gauche "Activités et Tâches", notre vue Kanban s'affiche par miracle :)

![Capture d’écran, le 2023-10-07 à 14 29 07](https://github.com/nuage365/Tutoriels/assets/102873102/35f4d317-a6c9-4567-9cda-bbe0f997ec63)

#### Etape 29 : On va tester la vue. On prend l'activité "Faire tutoriel sur Kanban" et on la glisse vers la colonne "Terminé". Un écran s'affiche pour confirmer le changement de statut ! On clique sur le bouton "Fermer Activités et Tâches" et l'activité est bien déplacée.

![Capture d’écran, le 2023-10-07 à 14 29 20](https://github.com/nuage365/Tutoriels/assets/102873102/0a36d300-0321-4fd1-87bb-42874d152b9a)

#### Etape 30 : On peut filtrer après sur ce que l'on veut comme la date d'échéance. Ici j'affiche les activités à venir des 7 prochains jours.
On peut ajuster certains éléments du Kanban lorsque nous sommes dans la définition de la vue au niveau du contrôle.

![Capture d’écran, le 2023-10-07 à 14 31 14](https://github.com/nuage365/Tutoriels/assets/102873102/743c2e2f-dfa7-46c7-a311-d2ede54fcfa7)

> Comme d'habitude, Si vous avez des questions ou besoin d'un consultant PowerApps / Dynamics, vous pouvez me contacter sur [Linkedin](https://www.linkedin.com/in/dominiquedelaire/)
> 
> Dominique












