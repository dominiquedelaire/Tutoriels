# Introduction à la personnalisation d'un modèle Hugging Face pour la Finance
> Version 2025.09.22.1, Auteur : Dominique Delaire, Neurones Solutions Inc.

![imgintro](https://github.com/user-attachments/assets/6eb81051-ab9d-4ea2-8fd2-adfd79d6452d)


L'objectif de cette introduction est de montrer comment **partir d’un modèle open source Hugging Face** et le spécialiser pour des cas financiers (analyse de sentiments, classification de documents comptables, prévisions, etc.).
La deuxième partie montrera un exemple concret de détection des anomalies dans des transactions comptables et financières.

## Pourquoi Hugging Face pour la Finance ?

Hugging Face est devenu la plateforme de référence pour :
- Explorer des milliers de modèles (transformers, LLMs, embeddings…)
- Partager des datasets
- Fine-tuner ou adapter des modèles existants

En finance, ces modèles peuvent servir à :
- **Analyser** des rapports financiers ou des transactions
- **Détecter** des anomalies ou risques
- **Prédire** l'évolution des revenus/dépenses
- **Automatiser** des résumés de bilans ou comptes de résultat


## Choisir un modèle de base

Deux approches sont possibles :
- **Modèles financiers spécialisés** déjà présents sur Hugging Face (ex. [FinBERT](https://huggingface.co/yiyanghkust/finbert-tone))
- **Modèles généralistes** (LLaMA, Mistral, Falcon) à adapter via fine-tuning ou RAG

Exemple : **FinBERT** est optimisé pour analyser le ton (positif, neutre, négatif) de phrases financières.


## Préparer ses données

Types de données utiles :
- Rapports annuels (PDF → texte)
- Journaux comptables
- Transactions historiques
- Données de marché

### Points importants :
- Nettoyer les données (pas de doublons, format clair)
- Sécuriser la confidentialité (critique en finance !)
- Structurer les datasets (CSV, JSON, Parquet)


## Fine-tuning vs RAG (Retrieval-Augmented Generation)

### Fine-tuning
- Permet d'adapter le modèle à vos propres données
- Ex. : entraîner sur des bilans canadiens/québécois

### RAG
- Utilise une **base vectorielle** (ChromaDB, Qdrant, Milvus)
- Le modèle reste généraliste mais retrouve l'info dans vos documents
- Avantage : mise à jour dynamique sans réentraînement

Dans un projet comme notre projet *Shellbots LedgerAI*, le RAG est souvent plus pratique que le fine-tuning.


## Exemple de code : utilisation de FinBERT

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification, pipeline

# Charger le modèle FinBERT
model_name = "yiyanghkust/finbert-tone"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSequenceClassification.from_pretrained(model_name)

nlp = pipeline("sentiment-analysis", model=model, tokenizer=tokenizer)

# Exemple d'analyse de texte financier
text = "The company's revenue increased by 25% this quarter."
result = nlp(text)
print(result)
```
**Sortie attendue :**
```json
[{'label': 'Positive', 'score': 0.998}]
```

## Exemple de RAG avec Hugging Face et Chroma DB
```python
from langchain.embeddings import HuggingFaceEmbeddings
from langchain.vectorstores import Chroma
from langchain.chains import RetrievalQA
from langchain.llms import HuggingFaceHub

# Embeddings Hugging Face
embeddings = HuggingFaceEmbeddings(model_name="sentence-transformers/all-MiniLM-L6-v2")

# Base vectorielle Chroma
db = Chroma(persist_directory="finance_db", embedding_function=embeddings)

retriever = db.as_retriever()

# Modèle Hugging Face hébergé
llm = HuggingFaceHub(repo_id="mistralai/Mistral-7B-Instruct-v0.1")

qa = RetrievalQA.from_chain_type(llm=llm, retriever=retriever)

query = "Quels sont les principaux risques dans le rapport annuel 2023 ?"
print(qa.run(query))
```

## Cas d’usage concret

- **Détection de risques** dans des transactions comptables  
- **Résumé automatique** d'un rapport trimestriel  
- **Génération de prévisions** de cashflow  
- **Assistance fiscale** (via GPT + RAG sur lois fiscales canadiennes/québécoises)  


## Conclusion et ouverture

- Hugging Face permet d'avoir une **base open source puissante**.  
- La personnalisation (fine-tuning ou RAG) transforme un modèle générique en **assistant financier spécialisé**.  
- Future direction : combiner avec des agents (CrewAI, LangChain, Shellbots LedgerAI) pour automatiser des processus complets.
- Le prochain tutoriel présentera un exemple concret sur la détection d'anomalies dans les transactions comptables et financières.


## Ressources utiles

- [Hugging Face Models](https://huggingface.co/models)  
- [FinBERT](https://huggingface.co/yiyanghkust/finbert-tone)  
- [LangChain Documentation](https://python.langchain.com)  
- [ChromaDB](https://www.trychroma.com/)  

<br>

> Comme d'habitude, Si vous avez des questions, vous pouvez me contacter directement sur [Linkedin](https://www.linkedin.com/in/dominiquedelaire/) ou commenter l'article directement de Linkedin dans les commentaires :)
> Dominique
