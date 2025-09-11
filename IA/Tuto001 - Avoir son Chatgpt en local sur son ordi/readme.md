# Tutoriel "Avoir son ChatGpt en local sur son ordi"
> Version 2025-09-10, Auteur : Dominique Delaire
> 

<img width="796" height="536" alt="Capture d’écran, le 2025-09-10 à 22 12 04" src="https://github.com/user-attachments/assets/0da56731-3c3c-49cd-995c-da149b7455e3" />

**Dans cet exemple, nous allons voir pas à pas comment faire une app style chatgpt avec n'importe quel modèle du marché sans coder une seule ligne.**

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







