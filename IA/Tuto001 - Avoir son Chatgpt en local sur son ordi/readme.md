# Tutoriel "Avoir son ChatGpt en local sur son ordi"
> Version 2025-09-10, Auteur : Dominique Delaire
> 

<img width="796" height="536" alt="Capture d’écran, le 2025-09-10 à 22 12 04" src="https://github.com/user-attachments/assets/0da56731-3c3c-49cd-995c-da149b7455e3" />

**Dans cet exemple, nous allons voir pas à pas comment faire une app style chatgpt avec n'importe quel modèle du marché sans coder une seule ligne. Un autre tuto vous montrera comment ajouter des données à un modèle existant**

Avec l'aide de l'outil opensource **Ollama de Meta**, nous allons pouvoir utiliser des modèles de OpenAI pour faire la **même chose que Chatgpt mais en local et sans avoir besoin d'internet :)**   
**Ollama** est un moteur local pour LLMs (Modèles de langage), simple à utiliser, sécurisé et personnalisable.

## Première étape : Installer Ollama 

Il suffit d'aller à l'adresse [https://ollama.com/download](https://ollama.com/download) pour télécharger Ollama pour Linux, Windows ou MacOS.

## Deuxième étape : Installer un modèle tel qu'un modèle ChatGpt d'OpenAi

Un des modèles intéressants de ChatGpt, l'équivalent de GPTo4 est **le modèle GPT-OSS-120B**   

- Il a été développé par OpenAI sous licence libre, et donc on peut le modifier, l'utiliser, ou le déployer commercialement sans restrictions particulières.
- Il excelle dans le raisonnement complexe
- Il suit des instructions précises

Si vous n'avez pas au moins une gpu de 80Go, vous pouvez opter pour un modèle plus light très puissant aussi le **GPT-OSS-20B**

Dans Ollama, pour installer un modèle il suffit de taper la commande suivante : **ollama pull nomdumodèle** Exemple : **ollama pull gpt-oss:120b** ou **ollama pull gpt-oss:20b** 

<img width="1101" height="155" alt="Capture d’écran, le 2025-09-10 à 20 59 32" src="https://github.com/user-attachments/assets/6b2db618-c08b-41cc-81ae-0c7a0e632339" />   

**Le modèle 120b fait environ 65Gb sur le disque.**
<img width="1101" height="159" alt="Capture d’écran, le 2025-09-10 à 21 28 27" src="https://github.com/user-attachments/assets/b7216425-ec71-43b3-8c49-e6273e2dde26" />

### Troisième étape : On vérifie que le modèle est bien installé

Pour cela, on tape la commande **Ollama list**

<img width="940" height="184" alt="Capture d’écran, le 2025-09-10 à 21 50 08" src="https://github.com/user-attachments/assets/1eb57319-95c2-4840-83ca-4da1c7c011b4" />

### Quatrième étape : On teste le modèle 

Soit dans l'interface terminal de Ollama, on lance la commande **Ollama run nomdumodèle**, soit on lance **l'interface graphique de Ollama**.

Ici l'interface de Ollama avec une liste de modèles :

<img width="796" height="600" alt="Capture d’écran, le 2025-09-10 à 21 55 50" src="https://github.com/user-attachments/assets/6d625a4a-fb62-4ccc-b547-28565fd65bc5" />

**Je choisis ici le modèle de OpenAI que j'ai précédemment installé et je lui pose une question comme si j'étais dans Chatgpt :**
<img width="796" height="600" alt="Capture d’écran, le 2025-09-10 à 21 58 13" src="https://github.com/user-attachments/assets/b80987ee-12c7-432a-867f-55c6592137dd" />
<img width="796" height="523" alt="Capture d’écran, le 2025-09-10 à 22 10 38" src="https://github.com/user-attachments/assets/bf881a10-d61f-4a92-a76c-540ac0f10d8f" />

