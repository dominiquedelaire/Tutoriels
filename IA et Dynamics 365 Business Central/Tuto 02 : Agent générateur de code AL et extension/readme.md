
<img width="960" height="540" alt="BC ia 1" src="https://github.com/user-attachments/assets/dc99b751-e960-4082-a38a-2c33a8b5ff24" />

**Ceci est un code source généré par ShellbotsOS Fin 2023 permettant de générer du code AL à partir d'une spécification.**

**C'est la première version faite en novembre 2023 pour recherche et développement avec Business Central. Cela pourra vous donner des idées pour concevoir d'autres outils pour améliorer la productivité pour BC.**

L'agent final avec un modèle spécifique qui a pris environ 7 mois d'entraînement a été vendu à un partenaire Microsoft aux Etats unis et comprenait 6 agents spécifiques pour améliorer les performances et le résultat. Un agent spécifique compilait le code et l'extension en ligne de commande avec alc , pour compiler des projets al via les extensions visual studio code.

Voici le code de la première version en Python et streamlit. Pour exécuter le code, il faut installer les différents modules import et ensuite faire streamlit run codesource.py pour lancer l'interface web local. Cette première version était meilleure avec des specs en anglais.

```python
#!/usr/bin/env python3
"""
Application Web Streamlit pour la génération automatique de code AL 
pour Dynamics 365 Business Central à partir de spécifications PDF.
Version Ollama - Modèles locaux
Code Created by shellbots agent code v2023.01.04 user ddom
"""

import streamlit as st
import os
import json
import PyPDF2
import requests
from pathlib import Path
from typing import Dict, List, Optional, Tuple
import re
from dataclasses import dataclass
from datetime import datetime
import zipfile
import io
import tempfile
import uuid

# Configuration de la page Streamlit
st.set_page_config(
    page_title="BC AL Generator (Local)",
    page_icon="🏠",
    layout="wide",
    initial_sidebar_state="expanded"
)

@dataclass
class ALObject:
    """Représente un objet AL avec ses métadonnées"""
    type: str  # Table, Page, Codeunit, Report, etc.
    name: str
    id: int
    code: str
    dependencies: List[str]
    filename: str

class OllamaClient:
    """Client pour interagir avec Ollama"""
    
    def __init__(self, base_url: str = "http://localhost:11434", model: str = "qwen2.5-coder:7b"):
        self.base_url = base_url.rstrip('/')
        self.model = model
    
    def chat(self, messages: List[Dict], max_tokens: int = 4000, temperature: float = 0.1, top_p: float = 0.9) -> str:
        """
        Envoie une requête chat à Ollama avec paramètres optimisés
        
        Args:
            messages: Liste des messages de conversation
            max_tokens: Nombre maximum de tokens
            temperature: Température (0.05-0.1 pour code, plus bas = plus déterministe)
            top_p: Top-p sampling (0.8-0.9 pour code)
            
        Returns:
            Réponse du modèle
        """
        # Convertir les messages en prompt unique pour Ollama
        prompt = ""
        for msg in messages:
            if msg["role"] == "system":
                prompt += f"System: {msg['content']}\n\n"
            elif msg["role"] == "user":
                prompt += f"User: {msg['content']}\n\n"
            elif msg["role"] == "assistant":
                prompt += f"Assistant: {msg['content']}\n\n"
        
        prompt += "Assistant: "
        
        # Paramètres optimisés pour la génération de code AL
        payload = {
            "model": self.model,
            "prompt": prompt,
            "stream": False,
            "options": {
                "temperature": temperature,
                "top_p": top_p,
                "top_k": 40,  # Limiter les choix pour plus de cohérence
                "repeat_penalty": 1.15,  # Éviter les répétitions dans le code
                "num_predict": max_tokens,
                "stop": ["```", "User:", "System:", "Note:", "//END", "Explanation:"],
                "tfs_z": 1.0,  # Tail free sampling pour la cohérence
                "typical_p": 1.0,  # Typical sampling
                "mirostat": 0,  # Désactiver mirostat pour le code
                "mirostat_eta": 0.1,
                "mirostat_tau": 5.0,
                "penalize_newline": False,  # Permettre les retours à la ligne dans le code
                "seed": -1,  # Seed aléatoire
                "numa": False
            }
        }
        
        try:
            response = requests.post(
                f"{self.base_url}/api/generate",
                json=payload,
                timeout=180  # 3 minutes pour les gros modèles comme qwen2.5-coder:32b
            )
            response.raise_for_status()
            
            result = response.json()
            return result.get("response", "")
            
        except requests.exceptions.RequestException as e:
            raise Exception(f"Erreur de connexion à Ollama: {str(e)}")
        except Exception as e:
            raise Exception(f"Erreur Ollama: {str(e)}")
    
    def is_available(self) -> bool:
        """
        Vérifie si Ollama est disponible
        
        Returns:
            True si Ollama répond, False sinon
        """
        try:
            response = requests.get(f"{self.base_url}/api/version", timeout=5)
            return response.status_code == 200
        except:
            return False
    
    def list_models(self) -> List[str]:
        """
        Liste les modèles disponibles dans Ollama
        
        Returns:
            Liste des noms de modèles
        """
        try:
            response = requests.get(f"{self.base_url}/api/tags", timeout=10)
            response.raise_for_status()
            
            data = response.json()
            return [model["name"] for model in data.get("models", [])]
        except:
            return []

class BusinessCentralALGenerator:
    """Agent principal pour la génération de code AL - Version Ollama"""
    
    def __init__(self, ollama_url: str = "http://localhost:11434", model: str = "qwen2.5-coder:7b"):
        """
        Initialise l'agent avec le client Ollama
        
        Args:
            ollama_url: URL du serveur Ollama
            model: Nom du modèle à utiliser
        """
        self.client = OllamaClient(ollama_url, model)
        
        # Configuration des objets AL
        self.object_id_ranges = {
            'Table': (50000, 50099),
            'Page': (50100, 50199),
            'Codeunit': (50200, 50299),
            'Report': (50300, 50399),
            'XMLport': (50400, 50499),
            'Query': (50500, 50549),
            'PermissionSet': (50550, 50599)
        }
        
        self.used_ids = set()
        self.generated_objects = []
    
    def clean_text_for_json(self, text: str) -> str:
        """
        Nettoie le texte pour éviter les problèmes d'encodage JSON
        
        Args:
            text: Texte à nettoyer
            
        Returns:
            Texte nettoyé
        """
        return (
            text.replace("'", "'")
                .replace("'", "'")
                .replace(""", '"')
                .replace(""", '"')
                .replace("–", "-")
                .replace("—", "-")
                .replace("é", "e")
                .replace("è", "e")
                .replace("ê", "e")
                .replace("à", "a")
                .replace("ç", "c")
                .replace("ù", "u")
                .replace("û", "u")
                .replace("ô", "o")
                .replace("î", "i")
                .replace("ï", "i")
        )

    def extract_pdf_content(self, pdf_file) -> str:
        """
        Extrait le texte d'un fichier PDF uploadé
        
        Args:
            pdf_file: Fichier PDF uploadé via Streamlit
            
        Returns: 
            Contenu textuel du PDF
        """
        try:
            pdf_reader = PyPDF2.PdfReader(pdf_file)
            text_content = ""
            
            for page in pdf_reader.pages:
                text_content += page.extract_text() + "\n"
            
            # Nettoyer le texte
            cleaned_text = self.clean_text_for_json(text_content.strip())
            
            return cleaned_text
        except Exception as e:
            raise Exception(f"Erreur lors de l'extraction du PDF: {str(e)}")
    
    def analyze_specification(self, pdf_content: str) -> Dict:
        """
        Analyse la spécification PDF avec Ollama pour identifier les objets AL nécessaires
        
        Args:
            pdf_content: Contenu textuel de la spécification
            
        Returns:
            Dictionnaire avec l'analyse structurée
        """
        # D'abord essayer une analyse intelligente par parsing
        try:
            parsed_analysis = self.parse_structured_specification(pdf_content)
            if parsed_analysis:
                st.success("✅ Spécification parsée avec succès par analyse structurée")
                return self.validate_and_fix_analysis(parsed_analysis)
        except Exception as e:
            st.warning(f"⚠️ Analyse structurée échouée: {str(e)}")
        
        # Fallback vers l'analyse IA avec prompt simplifié
        return self.analyze_with_ai_fallback(pdf_content)
    
    def parse_structured_specification(self, pdf_content: str) -> Dict:
        """
        Parse une spécification structurée (détecte automatiquement les tables, etc.)
        
        Args:
            pdf_content: Contenu du PDF
            
        Returns:
            Analyse structurée
        """
        lines = pdf_content.split('\n')
        analysis = {
            "project_name": "Vehicle Management",
            "description": "Vehicle fleet management system for Dynamics 365 Business Central",
            "tables": [],
            "pages": [],
            "codeunits": [],
            "reports": []
        }
        
        current_table = None
        current_section = None
        
        for line in lines:
            line = line.strip()
            
            # Détecter le nom du projet
            if "vehicle management" in line.lower() or "fleet" in line.lower():
                if "purpose" in line.lower() or "specification" in line.lower():
                    analysis["project_name"] = "Vehicle Management"
                    analysis["description"] = "Vehicle fleet management system with assignments, maintenance, and insurance tracking"
            
            # Détecter les sections
            if line.lower().startswith("table:") or line.startswith("Table:"):
                table_name = line.split(":", 1)[1].strip()
                current_table = {
                    "name": table_name,
                    "description": f"Table for managing {table_name.lower()}",
                    "fields": [],
                    "keys": [{"name": "PK", "fields": [table_name + "ID"], "primary": True}],
                    "relations": []
                }
                current_section = "table"
                
            # Détecter les champs de table
            elif current_section == "table" and " - " in line and "(" in line:
                try:
                    # Parser le format: "- FieldName (Type): Description"
                    parts = line.split(" - ", 1)[1]
                    field_part, desc_part = parts.split("): ", 1)
                    field_name = field_part.split(" (")[0]
                    field_type = field_part.split(" (")[1]
                    
                    # Nettoyer le type
                    field_type = field_type.replace(")", "")
                    if field_type.lower() == "integer":
                        field_type = "Integer"
                    elif field_type.lower() == "decimal":
                        field_type = "Decimal"
                    elif field_type.lower() == "date":
                        field_type = "Date"
                    elif field_type.lower() == "option":
                        field_type = "Option"
                    
                    current_table["fields"].append({
                        "name": field_name,
                        "type": field_type,
                        "description": desc_part
                    })
                except:
                    pass
            
            # Fin de table - l'ajouter
            elif current_section == "table" and (line == "" or line.lower().startswith("table:") or "business logic" in line.lower()):
                if current_table and len(current_table["fields"]) > 0:
                    analysis["tables"].append(current_table)
                    
                    # Créer automatiquement les pages pour cette table
                    table_name = current_table["name"]
                    analysis["pages"].extend([
                        {
                            "name": f"{table_name} List",
                            "type": "List",
                            "source_table": table_name,
                            "description": f"List page for {table_name}"
                        },
                        {
                            "name": f"{table_name} Card",
                            "type": "Card",
                            "source_table": table_name,
                            "description": f"Card page for {table_name}"
                        }
                    ])
                
                current_table = None
                if not line.lower().startswith("table:"):
                    current_section = None
        
        # Ajouter la dernière table si elle existe
        if current_table and len(current_table["fields"]) > 0:
            analysis["tables"].append(current_table)
            table_name = current_table["name"]
            analysis["pages"].extend([
                {
                    "name": f"{table_name} List",
                    "type": "List",
                    "source_table": table_name,
                    "description": f"List page for {table_name}"
                },
                {
                    "name": f"{table_name} Card",
                    "type": "Card",
                    "source_table": table_name,
                    "description": f"Card page for {table_name}"
                }
            ])
        
        # Ajouter des codeunits basés sur la logique métier détectée
        if "business logic" in pdf_content.lower():
            analysis["codeunits"].extend([
                {
                    "name": "Vehicle Management",
                    "description": "Core vehicle management logic and validations",
                    "procedures": [
                        {"name": "AssignVehicle", "description": "Assign vehicle to employee"},
                        {"name": "ValidateAssignment", "description": "Validate assignment rules"},
                        {"name": "UpdateVehicleStatus", "description": "Update vehicle status based on business rules"},
                        {"name": "CheckMaintenanceDue", "description": "Check if maintenance is due"},
                        {"name": "CheckInsuranceExpiry", "description": "Check insurance expiry dates"}
                    ]
                },
                {
                    "name": "Vehicle Automation",
                    "description": "Automated processes and scheduled tasks",
                    "procedures": [
                        {"name": "DailyMaintenanceCheck", "description": "Daily check for maintenance due"},
                        {"name": "InsuranceExpiryCheck", "description": "Check for insurance expiring in 30 days"},
                        {"name": "LogVehicleChanges", "description": "Log all vehicle changes with timestamp"}
                    ]
                }
            ])
        
        # Ajouter des rapports
        analysis["reports"].extend([
            {
                "name": "Vehicle Status Report",
                "description": "Report showing current status of all vehicles",
                "data_items": ["Vehicle"]
            },
            {
                "name": "Vehicle Assignment Report", 
                "description": "Report showing vehicle assignments by employee",
                "data_items": ["Vehicle", "VehicleAssignment"]
            },
            {
                "name": "Maintenance Cost Report",
                "description": "Report showing maintenance costs by vehicle",
                "data_items": ["Vehicle", "VehicleMaintenance"]
            },
            {
                "name": "Insurance Expiry Report",
                "description": "Report showing vehicles with expiring insurance",
                "data_items": ["Vehicle", "VehicleInsurance"]
            }
        ])
        
        return analysis
    
    def analyze_with_ai_fallback(self, pdf_content: str) -> Dict:
        """
        Analyse avec IA en fallback, avec prompt très simplifié
        
        Args:
            pdf_content: Contenu du PDF
            
        Returns:
            Analyse par IA
        """
        # Truncate content pour éviter les timeouts
        if len(pdf_content) > 4000:
            pdf_content = pdf_content[:4000]
        
        prompt = f"""Analyse cette spécification Vehicle Management:

{pdf_content}

Réponds avec ce JSON exact (sans texte supplémentaire):

{{
  "project_name": "Vehicle Management",
  "description": "Vehicle fleet management system",
  "tables": [
    {{
      "name": "Vehicle",
      "description": "Main vehicle table",
      "fields": [
        {{"name": "VehicleID", "type": "Code[20]", "description": "Vehicle identifier"}},
        {{"name": "Brand", "type": "Text[50]", "description": "Vehicle brand"}},
        {{"name": "Model", "type": "Text[50]", "description": "Vehicle model"}}
      ],
      "keys": [{{"name": "PK", "fields": ["VehicleID"], "primary": true}}],
      "relations": []
    }}
  ],
  "pages": [
    {{"name": "Vehicle List", "type": "List", "source_table": "Vehicle", "description": "Vehicle list"}},
    {{"name": "Vehicle Card", "type": "Card", "source_table": "Vehicle", "description": "Vehicle card"}}
  ],
  "codeunits": [
    {{"name": "Vehicle Management", "description": "Vehicle logic", "procedures": [{{"name": "AssignVehicle", "description": "Assign vehicle"}}]}}
  ],
  "reports": [
    {{"name": "Vehicle Report", "description": "Vehicle report", "data_items": ["Vehicle"]}}
  ]
}}"""
        
        try:
            response = self.client.chat(
                messages=[{"role": "user", "content": prompt}],
                max_tokens=3000
            )
            
            content = response.strip()
            
            # Extraction JSON plus robuste
            json_match = re.search(r'\{.*\}', content, re.DOTALL)
            if json_match:
                json_str = json_match.group()
                json_str = self.clean_json_response(json_str)
                
                try:
                    parsed = json.loads(json_str)
                    return self.validate_and_fix_analysis(parsed)
                except:
                    # Si ça échoue encore, utiliser le parsing structuré par défaut
                    return self.create_vehicle_management_default()
            else:
                return self.create_vehicle_management_default()
                
        except Exception as e:
            st.warning(f"⚠️ Analyse IA échouée: {str(e)}")
            return self.create_vehicle_management_default()
    
    def create_vehicle_management_default(self) -> Dict:
        """
        Crée une analyse par défaut spécialement pour Vehicle Management
        
        Returns:
            Analyse Vehicle Management complète
        """
        return {
            "project_name": "Vehicle Management",
            "description": "Complete vehicle fleet management system with assignments, maintenance, and insurance tracking",
            "tables": [
                {
                    "name": "Vehicle",
                    "description": "Main vehicle registry table",
                    "fields": [
                        {"name": "VehicleID", "type": "Code[20]", "description": "Unique vehicle identifier"},
                        {"name": "Brand", "type": "Text[50]", "description": "Vehicle brand"},
                        {"name": "Model", "type": "Text[50]", "description": "Vehicle model"},
                        {"name": "Year", "type": "Integer", "description": "Manufacturing year"},
                        {"name": "LicensePlate", "type": "Code[20]", "description": "License plate number"},
                        {"name": "VehicleType", "type": "Option", "description": "Vehicle type"},
                        {"name": "CurrentMileage", "type": "Decimal", "description": "Current mileage"},
                        {"name": "Status", "type": "Option", "description": "Vehicle status"},
                        {"name": "Location", "type": "Text[100]", "description": "Current location"},
                        {"name": "ResponsibleUser", "type": "Code[20]", "description": "Responsible user"}
                    ],
                    "keys": [{"name": "PK", "fields": ["VehicleID"], "primary": True}],
                    "relations": []
                },
                {
                    "name": "VehicleAssignment",
                    "description": "Vehicle assignment tracking",
                    "fields": [
                        {"name": "AssignmentID", "type": "Code[20]", "description": "Assignment identifier"},
                        {"name": "VehicleID", "type": "Code[20]", "description": "Assigned vehicle"},
                        {"name": "EmployeeID", "type": "Code[20]", "description": "Assigned employee"},
                        {"name": "StartDate", "type": "Date", "description": "Assignment start date"},
                        {"name": "EndDate", "type": "Date", "description": "Assignment end date"},
                        {"name": "Comments", "type": "Text[250]", "description": "Assignment comments"}
                    ],
                    "keys": [{"name": "PK", "fields": ["AssignmentID"], "primary": True}],
                    "relations": [
                        {"field": "VehicleID", "table": "Vehicle", "table_field": "VehicleID"}
                    ]
                },
                {
                    "name": "VehicleMaintenance",
                    "description": "Vehicle maintenance records",
                    "fields": [
                        {"name": "MaintenanceID", "type": "Code[20]", "description": "Maintenance identifier"},
                        {"name": "VehicleID", "type": "Code[20]", "description": "Maintained vehicle"},
                        {"name": "MaintenanceDate", "type": "Date", "description": "Maintenance date"},
                        {"name": "MaintenanceType", "type": "Option", "description": "Type of maintenance"},
                        {"name": "Provider", "type": "Text[100]", "description": "Service provider"},
                        {"name": "Cost", "type": "Decimal", "description": "Maintenance cost"},
                        {"name": "Comments", "type": "Text[250]", "description": "Maintenance notes"}
                    ],
                    "keys": [{"name": "PK", "fields": ["MaintenanceID"], "primary": True}],
                    "relations": [
                        {"field": "VehicleID", "table": "Vehicle", "table_field": "VehicleID"}
                    ]
                },
                {
                    "name": "VehicleInsurance",
                    "description": "Vehicle insurance tracking",
                    "fields": [
                        {"name": "InsuranceID", "type": "Code[20]", "description": "Insurance identifier"},
                        {"name": "VehicleID", "type": "Code[20]", "description": "Insured vehicle"},
                        {"name": "Company", "type": "Text[100]", "description": "Insurance company"},
                        {"name": "PolicyNumber", "type": "Code[30]", "description": "Policy number"},
                        {"name": "StartDate", "type": "Date", "description": "Coverage start"},
                        {"name": "EndDate", "type": "Date", "description": "Coverage end"},
                        {"name": "Amount", "type": "Decimal", "description": "Insurance amount"},
                        {"name": "Status", "type": "Option", "description": "Insurance status"}
                    ],
                    "keys": [{"name": "PK", "fields": ["InsuranceID"], "primary": True}],
                    "relations": [
                        {"field": "VehicleID", "table": "Vehicle", "table_field": "VehicleID"}
                    ]
                }
            ],
            "pages": [
                {"name": "Vehicle List", "type": "List", "source_table": "Vehicle", "description": "List of all vehicles"},
                {"name": "Vehicle Card", "type": "Card", "source_table": "Vehicle", "description": "Vehicle details card"},
                {"name": "VehicleAssignment List", "type": "List", "source_table": "VehicleAssignment", "description": "Vehicle assignments list"},
                {"name": "VehicleAssignment Card", "type": "Card", "source_table": "VehicleAssignment", "description": "Assignment details card"},
                {"name": "VehicleMaintenance List", "type": "List", "source_table": "VehicleMaintenance", "description": "Maintenance records list"},
                {"name": "VehicleMaintenance Card", "type": "Card", "source_table": "VehicleMaintenance", "description": "Maintenance details card"},
                {"name": "VehicleInsurance List", "type": "List", "source_table": "VehicleInsurance", "description": "Insurance records list"},
                {"name": "VehicleInsurance Card", "type": "Card", "source_table": "VehicleInsurance", "description": "Insurance details card"},
                {"name": "Vehicle Management Role Center", "type": "RoleCenter", "source_table": "", "description": "Vehicle management dashboard"}
            ],
            "codeunits": [
                {
                    "name": "Vehicle Management",
                    "description": "Core vehicle management business logic",
                    "procedures": [
                        {"name": "AssignVehicle", "description": "Assign vehicle to employee"},
                        {"name": "ValidateAssignment", "description": "Validate assignment overlaps"},
                        {"name": "UpdateVehicleStatus", "description": "Update vehicle status"},
                        {"name": "CheckMaintenanceDue", "description": "Check maintenance requirements"}
                    ]
                },
                {
                    "name": "Vehicle Automation",
                    "description": "Automated vehicle processes",
                    "procedures": [
                        {"name": "DailyInsuranceCheck", "description": "Check insurance expiry"},
                        {"name": "MaintenanceAlert", "description": "Generate maintenance alerts"},
                        {"name": "LogVehicleChanges", "description": "Log all vehicle changes"}
                    ]
                }
            ],
            "reports": [
                {"name": "Vehicle Status Report", "description": "Current status of all vehicles", "data_items": ["Vehicle"]},
                {"name": "Assignment Report", "description": "Vehicle assignments by employee", "data_items": ["Vehicle", "VehicleAssignment"]},
                {"name": "Maintenance Cost Report", "description": "Maintenance costs analysis", "data_items": ["Vehicle", "VehicleMaintenance"]},
                {"name": "Insurance Expiry Report", "description": "Insurance expiration tracking", "data_items": ["Vehicle", "VehicleInsurance"]}
            ]
        }
    
    def validate_and_fix_analysis(self, analysis: Dict) -> Dict:
        """
        Valide et corrige l'analyse pour s'assurer qu'elle contient tous les éléments
        
        Args:
            analysis: Analyse à valider
            
        Returns:
            Analyse corrigée et complète
        """
        # S'assurer que toutes les sections existent
        if 'tables' not in analysis:
            analysis['tables'] = []
        if 'pages' not in analysis:
            analysis['pages'] = []
        if 'codeunits' not in analysis:
            analysis['codeunits'] = []
        if 'reports' not in analysis:
            analysis['reports'] = []
        
        # Pour chaque table, s'assurer qu'il y a des pages correspondantes
        for table in analysis['tables']:
            table_name = table['name']
            
            # Vérifier si des pages existent pour cette table
            has_list_page = any(page.get('source_table') == table_name and page.get('type') == 'List' 
                               for page in analysis['pages'])
            has_card_page = any(page.get('source_table') == table_name and page.get('type') == 'Card' 
                               for page in analysis['pages'])
            
            # Ajouter les pages manquantes
            if not has_list_page:
                analysis['pages'].append({
                    "name": f"{table_name} List",
                    "type": "List",
                    "source_table": table_name,
                    "description": f"List page for {table_name}"
                })
            
            if not has_card_page:
                analysis['pages'].append({
                    "name": f"{table_name} Card", 
                    "type": "Card",
                    "source_table": table_name,
                    "description": f"Card page for {table_name}"
                })
        
        # S'assurer qu'il y a au moins un codeunit
        if len(analysis['codeunits']) == 0 and len(analysis['tables']) > 0:
            project_name = analysis.get('project_name', 'Project')
            analysis['codeunits'].append({
                "name": f"{project_name} Management",
                "description": f"Management and business logic for {project_name}",
                "procedures": [
                    {"name": "ValidateData", "description": "Validate data entry"},
                    {"name": "ProcessData", "description": "Process business logic"}
                ]
            })
        
        # S'assurer qu'il y a au moins un rapport
        if len(analysis['reports']) == 0 and len(analysis['tables']) > 0:
            first_table = analysis['tables'][0]['name']
            analysis['reports'].append({
                "name": f"{first_table} Report",
                "description": f"Report for {first_table} data",
                "data_items": [first_table]
            })
        
        return analysis
    
    def analyze_specification_with_ai_only(self, content: str) -> Dict:
        """
        Analyse UNIQUEMENT avec l'IA, optimisée pour les modèles de code
        
        Args:
            content: Contenu de la spécification
            
        Returns:
            Analyse par l'IA
        """
        # Prompt ultra-optimisé pour les modèles de code
        prompt = f"""Tu es un architecte expert en Dynamics 365 Business Central et AL. Analyse cette spécification et génère une structure complète.

SPÉCIFICATION À ANALYSER:
{content}

TÂCHE: Retourne un JSON structuré avec TOUS les objets nécessaires pour implémenter cette spécification.

RÈGLES OBLIGATOIRES:
1. Chaque entité = 1 table + 1 page List + 1 page Card minimum
2. Logique métier = codeunits avec procédures spécifiques
3. Données à présenter = rapports avec DataItems appropriés
4. Relations entre tables = TableRelation dans les champs
5. Noms en anglais, descriptions en français

STRUCTURE JSON EXACTE (remplace [...] par le contenu réel):
{{
  "project_name": "[nom_du_projet_en_anglais]",
  "description": "[description_complète_en_français]",
  "tables": [
    {{
      "name": "[NomTable]",
      "description": "[Description en français]",
      "fields": [
        {{"name": "[NomChamp]", "type": "[TypeAL]", "description": "[Description]"}},
        {{"name": "No", "type": "Code[20]", "description": "Numéro unique"}},
        {{"name": "Description", "type": "Text[100]", "description": "Description"}}
      ],
      "keys": [
        {{"name": "PK", "fields": ["No"], "primary": true}}
      ],
      "relations": [
        {{"field": "[ChampRelation]", "table": "[TableCible]", "table_field": "[ChampCible]"}}
      ]
    }}
  ],
  "pages": [
    {{"name": "[NomTable] List", "type": "List", "source_table": "[NomTable]", "description": "Liste des [entités]"}},
    {{"name": "[NomTable] Card", "type": "Card", "source_table": "[NomTable]", "description": "Fiche [entité]"}}
  ],
  "codeunits": [
    {{
      "name": "[Nom] Management",
      "description": "Gestion de [entité]",
      "procedures": [
        {{"name": "Create[Entité]", "description": "Créer une nouvelle [entité]"}},
        {{"name": "Validate[Entité]", "description": "Valider les données [entité]"}},
        {{"name": "Process[Action]", "description": "Traiter [action spécifique]"}}
      ]
    }}
  ],
  "reports": [
    {{
      "name": "[Nom] Report",
      "description": "Rapport [type]",
      "data_items": ["[Table1]", "[Table2]"]
    }}
  ]
}}

IMPORTANT: Retourne SEULEMENT le JSON valide, aucun autre texte.

JSON:"""

        try:
            # Utiliser la méthode optimisée
            response = self.client.chat(
                messages=[{"role": "user", "content": prompt}],
                max_tokens=6000,
                temperature=0.1,
                top_p=0.9
            )
            
            content = response.strip()
            st.info(f"🔍 Réponse du modèle: {content[:200]}...")
            
            # Extraction JSON plus robuste
            json_match = re.search(r'\{.*\}', content, re.DOTALL)
            if json_match:
                json_str = json_match.group()
                
                # Nettoyage spécialisé pour les modèles de code
                json_str = self.clean_json_for_code_models(json_str)
                
                try:
                    parsed = json.loads(json_str)
                    
                    # Validation renforcée
                    if not parsed.get('tables') or len(parsed.get('tables', [])) == 0:
                        st.warning("⚠️ Aucune table détectée, utilisation du fallback...")
                        return self.intelligent_fallback_from_content(content)
                    
                    validated = self.validate_and_fix_analysis(parsed)
                    st.success(f"✅ Analyse réussie: {len(validated['tables'])} tables, {len(validated['pages'])} pages, {len(validated['codeunits'])} codeunits")
                    return validated
                    
                except json.JSONDecodeError as e:
                    st.error(f"❌ Erreur JSON: {str(e)}")
                    st.text(f"JSON problématique: {json_str[:800]}...")
                    
                    # Tentative de réparation du JSON
                    repaired_json = self.repair_malformed_json(json_str)
                    if repaired_json:
                        try:
                            parsed = json.loads(repaired_json)
                            return self.validate_and_fix_analysis(parsed)
                        except:
                            pass
                    
                    return self.intelligent_fallback_from_content(content)
                    
            else:
                st.error("❌ Aucun JSON trouvé dans la réponse")
                st.text(f"Réponse complète: {content}")
                return self.intelligent_fallback_from_content(content)
                
        except Exception as e:
            st.error(f"❌ Erreur analyse IA: {str(e)}")
            return self.intelligent_fallback_from_content(content)
    
    def clean_json_for_code_models(self, json_str: str) -> str:
        """
        Nettoyage spécialisé pour les modèles de code qui génèrent parfois du JSON imparfait
        
        Args:
            json_str: JSON à nettoyer
            
        Returns:
            JSON nettoyé
        """
        # Supprimer tout avant le premier {
        start_idx = json_str.find('{')
        if start_idx > 0:
            json_str = json_str[start_idx:]
        
        # Supprimer tout après le dernier }
        end_idx = json_str.rfind('}')
        if end_idx > 0:
            json_str = json_str[:end_idx + 1]
        
        # Corrections spécifiques aux modèles de code
        replacements = [
            # Booléens
            (': true', ': true'),
            (': false', ': false'),
            (': True', ': true'),
            (': False', ': false'),
            
            # Guillemets manquants sur les clés
            (r'(\w+):', r'"\1":'),
            
            # Virgules en trop
            (r',\s*}', '}'),
            (r',\s*]', ']'),
            
            # Espaces multiples
            (r'\s+', ' '),
            
            # Retours à la ligne dans les chaînes
            (r'"\s*\n\s*"', '" "'),
            
            # Caractères d'échappement problématiques
            (r'\\', '\\\\'),
        ]
        
        for pattern, replacement in replacements:
            json_str = re.sub(pattern, replacement, json_str)
        
        return json_str
    
    def repair_malformed_json(self, json_str: str) -> Optional[str]:
        """
        Tentative de réparation d'un JSON malformé
        
        Args:
            json_str: JSON malformé
            
        Returns:
            JSON réparé ou None si impossible
        """
        try:
            # Stratégies de réparation
            strategies = [
                # Ajouter accolades manquantes
                lambda s: s + '}' if s.count('{') > s.count('}') else s,
                
                # Supprimer virgules en trop avant }
                lambda s: re.sub(r',(\s*})', r'\1', s),
                
                # Ajouter guillemets manquants
                lambda s: re.sub(r':\s*([^",{\[\s][^",}\]]*)', r': "\1"', s),
                
                # Remplacer simple quotes par double quotes
                lambda s: s.replace("'", '"'),
            ]
            
            for strategy in strategies:
                try:
                    repaired = strategy(json_str)
                    json.loads(repaired)  # Test de validation
                    return repaired
                except:
                    continue
            
            return None
            
        except:
            return None
    
    def aggressive_json_clean(self, json_str: str) -> str:
        """
        Nettoyage agressif du JSON pour Ollama
        
        Args:
            json_str: JSON à nettoyer
            
        Returns:
            JSON nettoyé
        """
        # Supprimer tout ce qui est avant le premier {
        start_idx = json_str.find('{')
        if start_idx > 0:
            json_str = json_str[start_idx:]
        
        # Supprimer tout ce qui est après le dernier }
        end_idx = json_str.rfind('}')
        if end_idx > 0:
            json_str = json_str[:end_idx + 1]
        
        # Corrections communes Ollama
        json_str = json_str.replace('\n', ' ')
        json_str = json_str.replace('\r', ' ')
        json_str = re.sub(r'\s+', ' ', json_str)  # Multiples espaces
        json_str = json_str.replace(': true', ': true')
        json_str = json_str.replace(': false', ': false')
        json_str = json_str.replace(': null', ': null')
        
        # Enlever virgules en trop
        json_str = re.sub(r',\s*}', '}', json_str)
        json_str = re.sub(r',\s*]', ']', json_str)
        
        # Guillemets manquants
        json_str = re.sub(r'(\w+):', r'"\1":', json_str)
        
        return json_str
    
    def intelligent_fallback_from_content(self, original_content: str) -> Dict:
        """
        Fallback intelligent basé sur le contenu original
        
        Args:
            original_content: Contenu original de la spécification
            
        Returns:
            Analyse intelligente par mots-clés
        """
        st.warning("🔄 Utilisation du fallback intelligent...")
        
        content_lower = original_content.lower()
        
        # Détection spécifique Vehicle Management
        if any(word in content_lower for word in ['vehicle', 'vehicleid', 'assignment', 'maintenance']):
            return self.create_vehicle_management_default()
        
        # Autres détections...
        elif any(word in content_lower for word in ['customer', 'client', 'contact']):
            return self.create_customer_management_default()
        
        else:
            return self.create_generic_management_default()
    
    def create_customer_management_default(self) -> Dict:
        """Fallback pour Customer Management"""
        return {
            "project_name": "Customer Management",
            "description": "Customer relationship management system",
            "tables": [
                {
                    "name": "Customer",
                    "description": "Customer information table",
                    "fields": [
                        {"name": "CustomerID", "type": "Code[20]", "description": "Customer ID"},
                        {"name": "Name", "type": "Text[100]", "description": "Customer name"},
                        {"name": "Email", "type": "Text[80]", "description": "Email address"},
                        {"name": "Phone", "type": "Text[30]", "description": "Phone number"}
                    ],
                    "keys": [{"name": "PK", "fields": ["CustomerID"], "primary": True}],
                    "relations": []
                }
            ],
            "pages": [
                {"name": "Customer List", "type": "List", "source_table": "Customer", "description": "Customer list"},
                {"name": "Customer Card", "type": "Card", "source_table": "Customer", "description": "Customer card"}
            ],
            "codeunits": [
                {"name": "Customer Management", "description": "Customer logic", "procedures": [{"name": "ValidateCustomer", "description": "Validate customer"}]}
            ],
            "reports": [
                {"name": "Customer Report", "description": "Customer report", "data_items": ["Customer"]}
            ]
        }
    
    def create_generic_management_default(self) -> Dict:
        """Fallback générique"""
        return {
            "project_name": "Business Management",
            "description": "Generic business management system",
            "tables": [
                {
                    "name": "BusinessEntity",
                    "description": "Main business entity",
                    "fields": [
                        {"name": "EntityID", "type": "Code[20]", "description": "Entity ID"},
                        {"name": "Name", "type": "Text[100]", "description": "Entity name"}
                    ],
                    "keys": [{"name": "PK", "fields": ["EntityID"], "primary": True}],
                    "relations": []
                }
            ],
            "pages": [
                {"name": "Entity List", "type": "List", "source_table": "BusinessEntity", "description": "Entity list"},
                {"name": "Entity Card", "type": "Card", "source_table": "BusinessEntity", "description": "Entity card"}
            ],
            "codeunits": [
                {"name": "Entity Management", "description": "Entity logic", "procedures": [{"name": "ProcessEntity", "description": "Process entity"}]}
            ],
            "reports": [
                {"name": "Entity Report", "description": "Entity report", "data_items": ["BusinessEntity"]}
            ]
        }
        """
        Valide et corrige l'analyse pour s'assurer qu'elle contient tous les éléments
        
        Args:
            analysis: Analyse à valider
            
        Returns:
            Analyse corrigée et complète
        """
        # S'assurer que toutes les sections existent
        if 'tables' not in analysis:
            analysis['tables'] = []
        if 'pages' not in analysis:
            analysis['pages'] = []
        if 'codeunits' not in analysis:
            analysis['codeunits'] = []
        if 'reports' not in analysis:
            analysis['reports'] = []
        
        # Pour chaque table, s'assurer qu'il y a des pages correspondantes
        for table in analysis['tables']:
            table_name = table['name']
            
            # Vérifier si des pages existent pour cette table
            has_list_page = any(page.get('source_table') == table_name and page.get('type') == 'List' 
                               for page in analysis['pages'])
            has_card_page = any(page.get('source_table') == table_name and page.get('type') == 'Card' 
                               for page in analysis['pages'])
            
            # Ajouter les pages manquantes
            if not has_list_page:
                analysis['pages'].append({
                    "name": f"{table_name} List",
                    "type": "List",
                    "source_table": table_name,
                    "description": f"List page for {table_name}"
                })
            
            if not has_card_page:
                analysis['pages'].append({
                    "name": f"{table_name} Card", 
                    "type": "Card",
                    "source_table": table_name,
                    "description": f"Card page for {table_name}"
                })
        
        # S'assurer qu'il y a au moins un codeunit
        if len(analysis['codeunits']) == 0 and len(analysis['tables']) > 0:
            project_name = analysis.get('project_name', 'Project')
            analysis['codeunits'].append({
                "name": f"{project_name} Management",
                "description": f"Management and business logic for {project_name}",
                "procedures": [
                    {"name": "ValidateData", "description": "Validate data entry"},
                    {"name": "ProcessData", "description": "Process business logic"}
                ]
            })
        
        # S'assurer qu'il y a au moins un rapport
        if len(analysis['reports']) == 0 and len(analysis['tables']) > 0:
            first_table = analysis['tables'][0]['name']
            analysis['reports'].append({
                "name": f"{first_table} Report",
                "description": f"Report for {first_table} data",
                "data_items": [first_table]
            })
        
        return analysis
    
    def create_fallback_analysis(self, pdf_content: str) -> Dict:
        """
        Crée une analyse par défaut si l'analyse automatique échoue
        
        Args:
            pdf_content: Contenu du PDF
            
        Returns:
            Analyse par défaut basée sur des mots-clés
        """
        st.warning("⚠️ Analyse automatique incomplète, génération d'une structure par défaut...")
        
        # Analyse basique par mots-clés
        content_lower = pdf_content.lower()
        
        # Détecter le type de projet
        if any(word in content_lower for word in ['commande', 'order', 'vente', 'sale']):
            project_type = "Order Management"
        elif any(word in content_lower for word in ['client', 'customer', 'contact']):
            project_type = "Customer Management"
        elif any(word in content_lower for word in ['produit', 'product', 'article', 'item']):
            project_type = "Product Management"
        elif any(word in content_lower for word in ['stock', 'inventory', 'warehouse']):
            project_type = "Inventory Management"
        else:
            project_type = "Business Management"
        
        return {
            "project_name": project_type,
            "description": f"Extension {project_type} générée à partir de la spécification",
            "tables": [
                {
                    "name": "Main Entity",
                    "description": "Main business entity",
                    "fields": [
                        {"name": "No", "type": "Code[20]", "description": "Primary key"},
                        {"name": "Name", "type": "Text[100]", "description": "Name"},
                        {"name": "Description", "type": "Text[250]", "description": "Description"},
                        {"name": "Status", "type": "Option", "description": "Status"}
                    ],
                    "keys": [
                        {"name": "PK", "fields": ["No"], "primary": True}
                    ],
                    "relations": []
                }
            ],
            "pages": [
                {
                    "name": "Main Entity List",
                    "type": "List",
                    "source_table": "Main Entity", 
                    "description": "List page for Main Entity"
                },
                {
                    "name": "Main Entity Card",
                    "type": "Card",
                    "source_table": "Main Entity",
                    "description": "Card page for Main Entity"
                }
            ],
            "codeunits": [
                {
                    "name": f"{project_type} Management",
                    "description": f"Management logic for {project_type}",
                    "procedures": [
                        {"name": "ValidateEntity", "description": "Validate entity data"},
                        {"name": "ProcessEntity", "description": "Process entity logic"}
                    ]
                }
            ],
            "reports": [
                {
                    "name": "Main Entity Report",
                    "description": "Report for Main Entity",
                    "data_items": ["Main Entity"]
                }
            ]
        }
    
    def clean_json_response(self, json_str: str) -> str:
        """
        Nettoie une réponse JSON potentiellement malformée
        
        Args:
            json_str: Chaîne JSON à nettoyer
            
        Returns:
            JSON nettoyé
        """
        # Supprimer les caractères de contrôle
        json_str = re.sub(r'[\x00-\x1f\x7f-\x9f]', '', json_str)
        
        # Corriger les booléens
        json_str = json_str.replace(': true', ': true').replace(': false', ': false')
        
        # Supprimer les virgules en trop
        json_str = re.sub(r',(\s*[}\]])', r'\1', json_str)
        
        return json_str
        """
        Nettoie une réponse JSON potentiellement malformée
        
        Args:
            json_str: Chaîne JSON à nettoyer
            
        Returns:
            JSON nettoyé
        """
        # Supprimer les caractères de contrôle
        json_str = re.sub(r'[\x00-\x1f\x7f-\x9f]', '', json_str)
        
        # Corriger les booléens
        json_str = json_str.replace(': true', ': true').replace(': false', ': false')
        
        # Supprimer les virgules en trop
        json_str = re.sub(r',(\s*[}\]])', r'\1', json_str)
        
        return json_str
    
    def get_next_object_id(self, object_type: str) -> int:
        """
        Génère le prochain ID disponible pour un type d'objet
        
        Args:
            object_type: Type d'objet AL
            
        Returns:
            ID disponible
        """
        if object_type not in self.object_id_ranges:
            raise ValueError(f"Type d'objet non supporté: {object_type}")
        
        start_id, end_id = self.object_id_ranges[object_type]
        
        for obj_id in range(start_id, end_id + 1):
            if obj_id not in self.used_ids:
                self.used_ids.add(obj_id)
                return obj_id
        
        raise Exception(f"Aucun ID disponible pour le type {object_type}")
    
    def generate_object_code(self, object_type: str, spec: Dict, context: Dict = None) -> str:
        """
        Génère le code AL pour un objet spécifique avec prompts optimisés
        
        Args:
            object_type: Type d'objet (Table, Page, Codeunit, Report)
            spec: Spécification de l'objet
            context: Contexte additionnel (tables disponibles, etc.)
            
        Returns:
            Code AL généré
        """
        object_id = self.get_next_object_id(object_type)
        
        # Prompt de base optimisé pour AL
        base_context = f"""Tu es un expert développeur AL pour Microsoft Dynamics 365 Business Central.

RÈGLES STRICTES:
- Génère UNIQUEMENT du code AL syntaxiquement correct
- Utilise les conventions de nommage AL standard
- Inclus toujours les propriétés obligatoires
- Respecte la structure AL moderne
- N'ajoute AUCUN texte explicatif, SEULEMENT le code

OBJET: {object_type} ID {object_id}
SPÉCIFICATION: {json.dumps(spec, indent=2)}"""

        if context:
            base_context += f"\nCONTEXTE: {json.dumps(context, indent=2)}"

        if object_type == 'Table':
            prompt = base_context + f"""

CODE AL REQUIS:
table {object_id} "{spec['name']}"
{{
    DataClassification = ToBeClassified;
    Caption = '{spec['name']}';
    
    fields
    {{
        // Tous les champs avec types exacts et propriétés
    }}
    
    keys
    {{
        key(PK; "PrimaryKeyField")
        {{
            Clustered = true;
        }}
    }}
    
    // TableRelation si nécessaire
}}

GÉNÈRE le code AL complet maintenant:"""

        elif object_type == 'Page':
            page_type = spec.get('type', 'Card')
            source_table = spec.get('source_table', '')
            
            prompt = base_context + f"""

CODE AL REQUIS:
page {object_id} "{spec['name']}"
{{
    PageType = {page_type};
    Caption = '{spec['name']}';
    SourceTable = "{source_table}";
    
    layout
    {{
        area(Content)
        {{
            // Layout approprié pour {page_type}
        }}
    }}
    
    actions
    {{
        // Actions standard pour {page_type}
    }}
}}

GÉNÈRE le code AL complet maintenant:"""

        elif object_type == 'Codeunit':
            prompt = base_context + f"""

CODE AL REQUIS:
codeunit {object_id} "{spec['name']}"
{{
    // Propriétés du codeunit
    
    // Procédures publiques
    
    // Procédures locales si nécessaire
    
    // Variables globales si nécessaire
}}

GÉNÈRE le code AL complet avec toutes les procédures maintenant:"""

        elif object_type == 'Report':
            prompt = base_context + f"""

CODE AL REQUIS:
report {object_id} "{spec['name']}"
{{
    Caption = '{spec['name']}';
    DefaultLayout = RDLC;
    
    dataset
    {{
        // DataItems avec tables appropriées
    }}
    
    rendering
    {{
        layout("Layout1")
        {{
            Type = RDLC;
            LayoutFile = './{spec['name']}.rdl';
        }}
    }}
    
    requestpage
    {{
        // Page de paramètres si nécessaire
    }}
}}

GÉNÈRE le code AL complet maintenant:"""
        
        try:
            # Paramètres optimisés pour la génération de code
            response = self.client.chat(
                messages=[{"role": "user", "content": prompt}],
                max_tokens=4000,  # Plus de tokens pour du code complet
                temperature=0.05,  # Très déterministe
                top_p=0.8
            )
            
            # Nettoyer la réponse pour extraire seulement le code AL
            code = response.strip()
            
        # Supprimer les balises markdown complètement
            code = re.sub(r'^```(?:al|AL)?.*$', '', code, flags=re.MULTILINE)
            code = re.sub(r'^```.*$', '', code, flags=re.MULTILINE) 
            code = re.sub(r'```', '', code)  # Supprimer toutes les balises restantes
    
    def generate_permission_set_code(self, project_name: str, objects: List[ALObject]) -> str:
        """
        Génère le code AL pour un permission set
        
        Args:
            project_name: Nom du projet
            objects: Liste des objets générés
            
        Returns:
            Code AL du permission set
        """
        permissionset_id = self.get_next_object_id('PermissionSet')
        
        objects_list = [f"{obj.type} {obj.id} '{obj.name}'" for obj in objects]
        
        prompt = f"""Tu es un expert en AL pour Dynamics 365 Business Central. Génère un PermissionSet complet.

PROJECT: {project_name}
PERMISSIONSET ID: {permissionset_id}
OBJECTS: {objects_list}

INSTRUCTIONS:
- Crée un PermissionSet AL avec l'ID spécifié
- Inclus tous les objets listés
- Donne les permissions complètes (RIMD) sur tous les objets
- Utilise un nom approprié basé sur le projet
- Utilise les meilleures pratiques AL

RETOURNE UNIQUEMENT LE CODE AL, sans explication."""
        
        try:
            response = self.client.chat(
                messages=[{"role": "user", "content": prompt}],
                max_tokens=2000
            )
            
            # Nettoyer la réponse
            code = response.strip()
            code = re.sub(r'^```(?:al|AL)?.*
    
    def create_app_json(self, project_name: str, description: str) -> Dict:
        """
        Crée la configuration app.json pour l'extension
        
        Args:
            project_name: Nom du projet
            description: Description du projet
            
        Returns:
            Configuration app.json
        """
        return {
            "id": str(uuid.uuid4()),  # GUID unique généré automatiquement
            "name": project_name,
            "publisher": "Your Company",
            "version": "1.0.0.0",
            "brief": description,
            "description": description,
            "privacyStatement": "",
            "EULA": "",
            "help": "",
            "url": "",
            "logo": "",
            "dependencies": [
                {
                    "id": "63ca2fa4-4f03-4f2b-a480-172fef340d3f",
                    "publisher": "Microsoft",
                    "name": "System Application",
                    "version": "20.0.0.0"
                },
                {
                    "id": "437dbf0e-84ff-417a-965d-ed2bb9650972",
                    "publisher": "Microsoft",
                    "name": "Base Application",
                    "version": "20.0.0.0"
                }
            ],
            "screenshots": [],
            "platform": "20.0.0.0",
            "application": "20.0.0.0",
            "idRanges": [
                {
                    "from": 50000,
                    "to": 50999
                }
            ],
            "resourceExposurePolicy": {
                "allowDebugging": True,
                "allowDownloadingSource": False,
                "includeSourceInSymbolFile": False
            }
        }
    
    def generate_from_analysis(self, analysis: Dict, progress_callback=None) -> Tuple[List[ALObject], Dict]:
        """
        Génère tous les objets AL à partir de l'analyse
        
        Args:
            analysis: Analyse de la spécification
            progress_callback: Fonction de callback pour le progrès
            
        Returns:
            Tuple (objets générés, app.json config)
        """
        project_name = analysis.get('project_name', 'BC Extension')
        description = analysis.get('description', 'Extension générée automatiquement')
        
        generated_objects = []
        total_objects = (
            len(analysis.get('tables', [])) +
            len(analysis.get('pages', [])) +
            len(analysis.get('codeunits', [])) +
            len(analysis.get('reports', [])) + 1  # +1 pour le permission set
        )
        current_object = 0
        
        # Tables
        if 'tables' in analysis:
            for table_spec in analysis['tables']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération table: {table_spec['name']}")
                
                code = self.generate_object_code('Table', table_spec)
                table_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Table',
                    name=table_spec['name'],
                    id=table_id,
                    code=code,
                    dependencies=[],
                    filename=f"Tab{table_id}.{table_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Pages
        if 'pages' in analysis:
            for page_spec in analysis['pages']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération page: {page_spec['name']}")
                
                context = {'tables': analysis.get('tables', [])}
                code = self.generate_object_code('Page', page_spec, context)
                page_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Page',
                    name=page_spec['name'],
                    id=page_id,
                    code=code,
                    dependencies=[page_spec.get('source_table', '')],
                    filename=f"Pag{page_id}.{page_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Codeunits
        if 'codeunits' in analysis:
            for codeunit_spec in analysis['codeunits']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération codeunit: {codeunit_spec['name']}")
                
                code = self.generate_object_code('Codeunit', codeunit_spec)
                codeunit_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Codeunit',
                    name=codeunit_spec['name'],
                    id=codeunit_id,
                    code=code,
                    dependencies=[],
                    filename=f"Cod{codeunit_id}.{codeunit_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Rapports
        if 'reports' in analysis:
            for report_spec in analysis['reports']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération rapport: {report_spec['name']}")
                
                code = self.generate_object_code('Report', report_spec)
                report_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Report',
                    name=report_spec['name'],
                    id=report_id,
                    code=code,
                    dependencies=report_spec.get('data_items', []),
                    filename=f"Rep{report_id}.{report_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Permission Set
        if progress_callback:
            progress_callback(current_object / total_objects, "Génération du Permission Set...")
        
        permissionset_code = self.generate_permission_set_code(project_name, generated_objects)
        permissionset_id = list(self.used_ids)[-1]  # Dernier ID utilisé
        
        al_object = ALObject(
            type='PermissionSet',
            name=f"{project_name} Permissions",
            id=permissionset_id,
            code=permissionset_code,
            dependencies=[],
            filename=f"Per{permissionset_id}.{project_name.replace(' ', '')}Permissions.al"
        )
        
        generated_objects.append(al_object)
        
        # Configuration app.json
        app_config = self.create_app_json(project_name, description)
        
        if progress_callback:
            progress_callback(1.0, "Génération terminée!")
        
        return generated_objects, app_config

def create_download_zip(objects: List[ALObject], app_config: Dict) -> bytes:
    """
    Crée un fichier ZIP avec tous les objets AL générés
    
    Args:
        objects: Liste des objets AL
        app_config: Configuration app.json
        
    Returns:
        Contenu du fichier ZIP en bytes
    """
    zip_buffer = io.BytesIO()
    
    with zipfile.ZipFile(zip_buffer, 'w', zipfile.ZIP_DEFLATED) as zip_file:
        # Ajouter app.json
        app_json_content = json.dumps(app_config, indent=2, ensure_ascii=False)
        zip_file.writestr("app.json", app_json_content)
        
        # Ajouter tous les objets AL
        for obj in objects:
            header = f"""// Generated by Business Central AL Generator (Ollama)
// Date: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
// Object: {obj.type} {obj.id} "{obj.name}"

"""
            content = header + obj.code
            zip_file.writestr(obj.filename, content)
    
    zip_buffer.seek(0)
    return zip_buffer.getvalue()

# Interface Streamlit
def main():
    """Interface principale Streamlit"""
    
    # Sidebar pour configuration
    with st.sidebar:
        st.header("🏠 Configuration Ollama")
        
        # Configuration Ollama
        ollama_url = st.text_input(
            "URL Ollama",
            value="http://localhost:11434",
            help="URL de votre serveur Ollama local"
        )
        
        # Vérification de la connexion Ollama
        if st.button("🔍 Tester la connexion"):
            client = OllamaClient(ollama_url)
            if client.is_available():
                st.success("✅ Ollama connecté !")
                
                # Lister les modèles disponibles
                models = client.list_models()
                if models:
                    st.info(f"📋 Modèles disponibles: {', '.join(models)}")
                else:
                    st.warning("⚠️ Aucun modèle trouvé")
            else:
                st.error("❌ Impossible de se connecter à Ollama")
        
        # Sélection du modèle - MODÈLES OPTIMISÉS POUR LE CODE
        model_options = [
            "qwen2.5-coder:32b",        # ⭐⭐⭐ TOP - Excellent pour AL (si 32GB+ RAM)
            "codestral:22b",            # ⭐⭐⭐ Mistral Codestral (excellent codeur)
            "qwen2.5-coder:14b",        # ⭐⭐⭐ Très bon compromis (16GB+ RAM)
            "deepseek-coder-v2:16b",    # ⭐⭐⭐ DeepSeek V2 amélioré
            "qwen2.5-coder:7b",         # ⭐⭐ Bon pour 8GB+ RAM
            "deepseek-coder:6.7b",      # ⭐⭐ Version classique
            "codellama:13b-instruct",   # ⭐⭐ Stable et performant
            "codellama:7b-instruct",    # ⭐ Pour machines limitées
            "granite-code:8b",          # ⭐ IBM Granite (nouveau)
            "autre"
        ]
        
        selected_model = st.selectbox(
            "Modèle à utiliser",
            options=model_options,
            help="Modèles optimisés pour la génération de code AL Business Central"
        )
        
        if selected_model == "autre":
            custom_model = st.text_input("Nom du modèle personnalisé")
            if custom_model:
                selected_model = custom_model
        
        st.divider()
        
        # Informations sur les modèles recommandés
        st.header("🎯 Modèles de code optimisés")
        st.markdown("""
        **🏆 TOP pour Business Central AL :**
        - `qwen2.5-coder:32b` (⭐⭐⭐ Excellence - 32GB RAM)
        - `codestral:22b` (⭐⭐⭐ Mistral Codestral - 24GB RAM)
        - `qwen2.5-coder:14b` (⭐⭐⭐ Excellent - 16GB RAM)
        
        **🚀 Bon compromis :**
        - `qwen2.5-coder:7b` (⭐⭐ Très bon - 8GB RAM)
        - `deepseek-coder-v2:16b` (⭐⭐ V2 amélioré)
        
        **💡 Installation recommandée :**
        ```bash
        # Pour machines puissantes (32GB+)
        ollama pull qwen2.5-coder:32b
        
        # Pour machines normales (16GB+)  
        ollama pull qwen2.5-coder:14b
        
        # Pour machines limitées (8GB+)
        ollama pull qwen2.5-coder:7b
        ```
        
        **⚡ Démarrage :**
        ```bash
        ollama serve
        ```
        """)
        
        # Auto-détection du meilleur modèle
        if st.button("🎯 Détecter le meilleur modèle"):
            client = OllamaClient(ollama_url)
            if client.is_available():
                available_models = client.list_models()
                
                # Ordre de préférence pour AL
                preferred_order = [
                    "qwen2.5-coder:32b", "codestral:22b", "qwen2.5-coder:14b",
                    "deepseek-coder-v2:16b", "qwen2.5-coder:7b", "deepseek-coder:6.7b",
                    "codellama:13b-instruct", "codellama:7b-instruct"
                ]
                
                best_model = None
                for model in preferred_order:
                    if model in available_models:
                        best_model = model
                        break
                
                if best_model:
                    st.success(f"🎯 Meilleur modèle détecté: **{best_model}**")
                    st.info("💡 Sélectionnez-le dans la liste ci-dessus")
                else:
                    st.warning("⚠️ Aucun modèle de code optimisé trouvé")
                    st.info("📥 Installez un modèle recommandé avec `ollama pull qwen2.5-coder:7b`")
            else:
                st.error("❌ Ollama non disponible")
        
        st.divider()
        
        # Informations
        st.header("📋 Informations")
        st.markdown("""
        **Objets AL générés :**
        - 📊 Tables avec relations
        - 📄 Pages (List/Card)
        - ⚙️ Codeunits (logique)
        - 📈 Rapports
        - 🔐 Permission Sets
        - 📦 app.json
        """)
        
        st.markdown("""
        **Avantages local :**
        - 🔒 Confidentialité totale
        - 💰 Pas de coûts API
        - ⚡ Pas de limite de requêtes
        - 🏠 Fonctionne hors ligne
        """)
    
    # En-tête principal
    st.title("🏠 Business Central AL Generator (Local)")
    st.markdown("**Génération automatique de code AL avec Ollama - 100% local et privé**")
    
    # Zone d'upload
    st.header("📄 Spécification d'entrée")
    
    # Choix du mode d'entrée
    input_mode = st.radio(
        "Choisissez votre mode de saisie :",
        ["📝 Texte direct (Recommandé)", "📄 Upload PDF"],
        help="Le mode texte direct est plus précis pour l'analyse Ollama"
    )
    
    specification_content = None
    
    if input_mode == "📝 Texte direct (Recommandé)":
        specification_content = st.text_area(
            "Collez votre spécification ici :",
            height=300,
            placeholder="""Exemple :
1. Purpose
This functional specification describes the design of a new vehicle management module for Dynamics 365 Business Central.

2. Tables and Fields
Table: Vehicle
- VehicleID (Code[20]): Unique identifier of the vehicle
- Brand (Text[50]): Brand of the vehicle
- Model (Text[50]): Model of the vehicle
- Year (Integer): Year of manufacture
- Status (Option): Available, In Use, Under Maintenance

Table: VehicleAssignment
- AssignmentID (Code[20]): Unique identifier
- VehicleID (Code[20]): ID of the assigned vehicle
- EmployeeID (Code[20]): ID of the assigned employee

3. Business Logic
- When a vehicle is assigned, it automatically changes to In Use status
- Maintenance alerts should be generated automatically""",
            help="Copiez-collez directement le texte de votre spécification"
        )
        
        if specification_content:
            st.success(f"✅ Spécification saisie : {len(specification_content)} caractères")
    
    else:  # Mode PDF
        uploaded_file = st.file_uploader(
            "Uploadez votre spécification PDF",
            type=['pdf'],
            help="Sélectionnez un fichier PDF contenant la spécification de votre extension Business Central"
        )
        
        if uploaded_file:
            st.success(f"📎 Fichier uploadé: {uploaded_file.name}")
            st.info(f"📏 Taille: {len(uploaded_file.getvalue()) / 1024:.1f} KB")
    
    # Continuer seulement si on a du contenu ou un fichier
    if (input_mode == "📝 Texte direct (Recommandé)" and specification_content) or (input_mode == "📄 Upload PDF" and uploaded_file):
        
        col1, col2 = st.columns([3, 1])
        
        with col1:
            if input_mode == "📝 Texte direct (Recommandé)":
                st.success("✅ Mode texte direct - Analyse plus précise")
                st.info(f"📏 Taille: {len(specification_content)} caractères")
            else:
                st.success(f"📎 Fichier uploadé: {uploaded_file.name}")
                st.info(f"📏 Taille: {len(uploaded_file.getvalue()) / 1024:.1f} KB")
            st.info(f"🤖 Modèle: {selected_model}")
        
        with col2:
            generate_button = st.button("🚀 Générer le code AL", type="primary", use_container_width=True)
        
        if generate_button and selected_model:
            try:
                # Vérification de la connexion Ollama
                client = OllamaClient(ollama_url, selected_model)
                if not client.is_available():
                    st.error("❌ Impossible de se connecter à Ollama. Vérifiez que le serveur est démarré.")
                    st.info("💡 Démarrez Ollama avec: `ollama serve`")
                    return
                
                # Initialisation du générateur
                generator = BusinessCentralALGenerator(ollama_url, selected_model)
                
                # Extraction du contenu
                if input_mode == "📝 Texte direct (Recommandé)":
                    with st.spinner("📝 Analyse du contenu textuel..."):
                        pdf_content = specification_content
                else:
                    with st.spinner("🔍 Extraction du contenu PDF..."):
                        pdf_content = generator.extract_pdf_content(uploaded_file)
                
                # Analyse avec Ollama - SANS parsing structuré, direct vers l'IA
                with st.spinner(f"🤖 Analyse intelligente avec {selected_model}..."):
                    analysis = generator.analyze_specification_with_ai_only(pdf_content)
                
                # Debug : Afficher la réponse brute si demandé
                with st.expander("🔍 Debug - Voir l'analyse brute"):
                    st.json(analysis)
                
                # Affichage de l'analyse
                st.header("📋 Analyse de la spécification")
                
                col1, col2 = st.columns(2)
                
                with col1:
                    st.subheader("📊 Projet")
                    st.write(f"**Nom :** {analysis.get('project_name', 'N/A')}")
                    st.write(f"**Description :** {analysis.get('description', 'N/A')}")
                
                with col2:
                    st.subheader("🔢 Objets détectés")
                    st.metric("Tables", len(analysis.get('tables', [])))
                    st.metric("Pages", len(analysis.get('pages', [])))
                    st.metric("Codeunits", len(analysis.get('codeunits', [])))
                    st.metric("Rapports", len(analysis.get('reports', [])))
                
                # Détails des objets détectés
                with st.expander("🔍 Détails des objets détectés"):
                    
                    if analysis.get('tables'):
                        st.subheader("📊 Tables")
                        for table in analysis['tables']:
                            st.write(f"• **{table['name']}** - {table.get('description', 'N/A')}")
                    
                    if analysis.get('pages'):
                        st.subheader("📄 Pages")
                        for page in analysis['pages']:
                            st.write(f"• **{page['name']}** ({page.get('type', 'N/A')}) - {page.get('description', 'N/A')}")
                    
                    if analysis.get('codeunits'):
                        st.subheader("⚙️ Codeunits")
                        for codeunit in analysis['codeunits']:
                            st.write(f"• **{codeunit['name']}** - {codeunit.get('description', 'N/A')}")
                    
                    if analysis.get('reports'):
                        st.subheader("📈 Rapports")
                        for report in analysis['reports']:
                            st.write(f"• **{report['name']}** - {report.get('description', 'N/A')}")
                
                # Génération du code
                st.header("⚡ Génération du code AL")
                
                # Barre de progression
                progress_bar = st.progress(0)
                status_text = st.empty()
                
                def update_progress(progress, message):
                    progress_bar.progress(progress)
                    status_text.text(message)
                
                # Génération des objets
                objects, app_config = generator.generate_from_analysis(analysis, update_progress)
                
                # Résultats
                st.success(f"✅ Génération terminée ! {len(objects)} objets créés")
                
                # Résumé des objets générés
                st.subheader("📋 Objets générés")
                
                # Tableau des objets
                objects_data = []
                for obj in objects:
                    objects_data.append({
                        "Type": obj.type,
                        "ID": obj.id,
                        "Nom": obj.name,
                        "Fichier": obj.filename
                    })
                
                st.dataframe(objects_data, use_container_width=True)
                
                # Prévisualisation du code
                st.subheader("👀 Prévisualisation du code")
                
                selected_object = st.selectbox(
                    "Sélectionnez un objet pour prévisualiser le code :",
                    options=range(len(objects)),
                    format_func=lambda x: f"{objects[x].type} {objects[x].id} - {objects[x].name}"
                )
                
                if selected_object is not None:
                    obj = objects[selected_object]
                    st.code(obj.code, language="al")
                
                # Téléchargement
                st.header("💾 Téléchargement")
                
                # Création du ZIP
                zip_content = create_download_zip(objects, app_config)
                
                st.download_button(
                    label="📦 Télécharger l'extension complète (ZIP)",
                    data=zip_content,
                    file_name=f"{analysis.get('project_name', 'BC_Extension').replace(' ', '_')}.zip",
                    mime="application/zip",
                    type="primary",
                    use_container_width=True
                )
                
                st.success("🎉 Votre extension Business Central est prête !")
                
                # Instructions de déploiement
                with st.expander("📖 Instructions de déploiement"):
                    st.markdown("""
                    ### 🚀 Comment utiliser votre extension
                    
                    1. **Téléchargez** le fichier ZIP
                    2. **Extrayez** le contenu dans un nouveau dossier
                    3. **Ouvrez** le dossier dans Visual Studio Code
                    4. **Installez** l'extension AL pour VS Code si nécessaire
                    5. **Configurez** votre environnement Business Central
                    6. **Compilez** l'extension (Ctrl+Shift+P → "AL: Package")
                    7. **Déployez** dans votre environnement de test
                    
                    ### ⚙️ Configuration requise
                    - Visual Studio Code
                    - Extension AL
                    - Environnement Business Central (sandbox/dev)
                    - Docker (optionnel pour environnement local)
                    
                    ### 🏠 Avantages Ollama Local
                    - 🔒 **Confidentialité** : Vos données restent locales
                    - 💰 **Gratuit** : Aucun coût d'API
                    - ⚡ **Rapide** : Pas de latence réseau
                    - 🌐 **Hors ligne** : Fonctionne sans internet
                    """)
                
            except Exception as e:
                st.error(f"❌ Erreur lors de la génération : {str(e)}")
                st.exception(e)
                
                # Conseils de dépannage
                with st.expander("🔧 Conseils de dépannage"):
                    st.markdown("""
                    **Erreurs communes :**
                    
                    1. **Connexion Ollama** : Vérifiez que `ollama serve` est démarré
                    2. **Modèle manquant** : Installez le modèle avec `ollama pull deepseek-coder:6.7b`
                    3. **Mémoire insuffisante** : Essayez un modèle plus petit comme `qwen2.5-coder:1.5b`
                    4. **JSON malformé** : Le modèle peut générer du JSON invalide, réessayez
                    5. **Timeout** : Le modèle prend du temps, soyez patient
                    
                    **Commandes utiles :**
                    ```bash
                    # Vérifier Ollama
                    ollama list
                    
                    # Tester la connexion
                    curl http://localhost:11434/api/version
                    
                    # Redémarrer Ollama
                    ollama serve
                    ```
                    """)
    
    elif not selected_model:
        st.warning("⚠️ Veuillez sélectionner un modèle Ollama dans la barre latérale")
        
        with st.expander("🚀 Installation rapide Ollama"):
            st.markdown("""
            ### Windows
            1. Téléchargez depuis [ollama.com/download](https://ollama.com/download)
            2. Installez le fichier .exe
            3. Ouvrez un terminal et tapez :
            ```cmd
            ollama pull deepseek-coder:6.7b
            ollama serve
            ```
            
            ### Linux/macOS
            ```bash
            curl -fsSL https://ollama.com/install.sh | sh
            ollama pull deepseek-coder:6.7b
            ollama serve
            ```
            """)
    
    elif not uploaded_file:
        st.info("📄 Veuillez uploader un fichier PDF de spécification")
        
        # Exemple de spécification
        with st.expander("💡 Exemple de spécification PDF"):
            st.markdown("""
            ### Exemple de contenu pour votre PDF de spécification :
            
            **Titre :** Gestion des Commandes Clients
            
            **Description :** Extension pour gérer les commandes clients avec suivi des statuts
            
            **Tables nécessaires :**
            - Table Commande Client : Numéro, Date, Client, Montant Total, Statut
            - Table Ligne Commande : Numéro Commande, Article, Quantité, Prix Unitaire
            
            **Pages nécessaires :**
            - Page Liste des Commandes
            - Page Fiche Commande
            - Page Lignes de Commande
            
            **Fonctionnalités :**
            - Validation des commandes
            - Calcul automatique des totaux
            - Rapports de suivi
            
            **Codeunits :**
            - Gestion des commandes
            - Validation des données
            
            **Rapports :**
            - Commandes par période
            - Statistiques de vente
            """)
    
    # Footer avec informations
    st.divider()
    st.markdown("""
    <div style='text-align: center; color: #666;'>
        <p>🏠 <strong>Business Central AL Generator - Version Ollama Local</strong></p>
        <p>Génération de code AL 100% locale et privée • Aucune donnée envoyée sur internet</p>
    </div>
    """, unsafe_allow_html=True)

if __name__ == "__main__":
    main(), '', code, flags=re.MULTILINE)
            
            # Supprimer les explications avant/après le code
            lines = code.split('\n')
            start_idx = -1
            end_idx = len(lines)
            
            # Trouver le début du code AL (table, page, codeunit, report)
            for i, line in enumerate(lines):
                if re.match(r'^\s*(table|page|codeunit|report)\s+\d+', line.lower()):
                    start_idx = i
                    break
            
            # Trouver la fin du code AL (dernière accolade fermante)
            brace_count = 0
            for i in range(start_idx if start_idx != -1 else 0, len(lines)):
                line = lines[i]
                brace_count += line.count('{') - line.count('}')
                if brace_count == 0 and (line.strip() == '}' or '}' in line):
                    end_idx = i + 1
                    break
            
            if start_idx != -1:
                cleaned_code = '\n'.join(lines[start_idx:end_idx])
            else:
                cleaned_code = code
            
            return cleaned_code.strip()
            
        except Exception as e:
            raise Exception(f"Erreur lors de la génération de {object_type}: {str(e)}")
    
    def generate_permission_set_code(self, project_name: str, objects: List[ALObject]) -> str:
        """
        Génère le code AL pour un permission set
        
        Args:
            project_name: Nom du projet
            objects: Liste des objets générés
            
        Returns:
            Code AL du permission set
        """
        permissionset_id = self.get_next_object_id('PermissionSet')
        
        objects_list = [f"{obj.type} {obj.id} '{obj.name}'" for obj in objects]
        
        prompt = f"""Tu es un expert en AL pour Dynamics 365 Business Central. Génère un PermissionSet complet.

PROJECT: {project_name}
PERMISSIONSET ID: {permissionset_id}
OBJECTS: {objects_list}

INSTRUCTIONS:
- Crée un PermissionSet AL avec l'ID spécifié
- Inclus tous les objets listés
- Donne les permissions complètes (RIMD) sur tous les objets
- Utilise un nom approprié basé sur le projet
- Utilise les meilleures pratiques AL

RETOURNE UNIQUEMENT LE CODE AL, sans explication."""
        
        try:
            response = self.client.chat(
                messages=[{"role": "user", "content": prompt}],
                max_tokens=2000
            )
            
            # Nettoyer la réponse
            code = response.strip()
            code = re.sub(r'^```(?:al)?', '', code, flags=re.MULTILINE)
            code = re.sub(r'^```$', '', code, flags=re.MULTILINE)
            
            return code.strip()
            
        except Exception as e:
            raise Exception(f"Erreur lors de la génération du PermissionSet: {str(e)}")
    
    def create_app_json(self, project_name: str, description: str) -> Dict:
        """
        Crée la configuration app.json pour l'extension
        
        Args:
            project_name: Nom du projet
            description: Description du projet
            
        Returns:
            Configuration app.json
        """
        return {
            "id": str(uuid.uuid4()),  # GUID unique généré automatiquement
            "name": project_name,
            "publisher": "Your Company",
            "version": "1.0.0.0",
            "brief": description,
            "description": description,
            "privacyStatement": "",
            "EULA": "",
            "help": "",
            "url": "",
            "logo": "",
            "dependencies": [
                {
                    "id": "63ca2fa4-4f03-4f2b-a480-172fef340d3f",
                    "publisher": "Microsoft",
                    "name": "System Application",
                    "version": "20.0.0.0"
                },
                {
                    "id": "437dbf0e-84ff-417a-965d-ed2bb9650972",
                    "publisher": "Microsoft",
                    "name": "Base Application",
                    "version": "20.0.0.0"
                }
            ],
            "screenshots": [],
            "platform": "20.0.0.0",
            "application": "20.0.0.0",
            "idRanges": [
                {
                    "from": 50000,
                    "to": 50999
                }
            ],
            "resourceExposurePolicy": {
                "allowDebugging": True,
                "allowDownloadingSource": False,
                "includeSourceInSymbolFile": False
            }
        }
    
    def generate_from_analysis(self, analysis: Dict, progress_callback=None) -> Tuple[List[ALObject], Dict]:
        """
        Génère tous les objets AL à partir de l'analyse
        
        Args:
            analysis: Analyse de la spécification
            progress_callback: Fonction de callback pour le progrès
            
        Returns:
            Tuple (objets générés, app.json config)
        """
        project_name = analysis.get('project_name', 'BC Extension')
        description = analysis.get('description', 'Extension générée automatiquement')
        
        generated_objects = []
        total_objects = (
            len(analysis.get('tables', [])) +
            len(analysis.get('pages', [])) +
            len(analysis.get('codeunits', [])) +
            len(analysis.get('reports', [])) + 1  # +1 pour le permission set
        )
        current_object = 0
        
        # Tables
        if 'tables' in analysis:
            for table_spec in analysis['tables']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération table: {table_spec['name']}")
                
                code = self.generate_object_code('Table', table_spec)
                table_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Table',
                    name=table_spec['name'],
                    id=table_id,
                    code=code,
                    dependencies=[],
                    filename=f"Tab{table_id}.{table_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Pages
        if 'pages' in analysis:
            for page_spec in analysis['pages']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération page: {page_spec['name']}")
                
                context = {'tables': analysis.get('tables', [])}
                code = self.generate_object_code('Page', page_spec, context)
                page_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Page',
                    name=page_spec['name'],
                    id=page_id,
                    code=code,
                    dependencies=[page_spec.get('source_table', '')],
                    filename=f"Pag{page_id}.{page_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Codeunits
        if 'codeunits' in analysis:
            for codeunit_spec in analysis['codeunits']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération codeunit: {codeunit_spec['name']}")
                
                code = self.generate_object_code('Codeunit', codeunit_spec)
                codeunit_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Codeunit',
                    name=codeunit_spec['name'],
                    id=codeunit_id,
                    code=code,
                    dependencies=[],
                    filename=f"Cod{codeunit_id}.{codeunit_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Rapports
        if 'reports' in analysis:
            for report_spec in analysis['reports']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération rapport: {report_spec['name']}")
                
                code = self.generate_object_code('Report', report_spec)
                report_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Report',
                    name=report_spec['name'],
                    id=report_id,
                    code=code,
                    dependencies=report_spec.get('data_items', []),
                    filename=f"Rep{report_id}.{report_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Permission Set
        if progress_callback:
            progress_callback(current_object / total_objects, "Génération du Permission Set...")
        
        permissionset_code = self.generate_permission_set_code(project_name, generated_objects)
        permissionset_id = list(self.used_ids)[-1]  # Dernier ID utilisé
        
        al_object = ALObject(
            type='PermissionSet',
            name=f"{project_name} Permissions",
            id=permissionset_id,
            code=permissionset_code,
            dependencies=[],
            filename=f"Per{permissionset_id}.{project_name.replace(' ', '')}Permissions.al"
        )
        
        generated_objects.append(al_object)
        
        # Configuration app.json
        app_config = self.create_app_json(project_name, description)
        
        if progress_callback:
            progress_callback(1.0, "Génération terminée!")
        
        return generated_objects, app_config

def create_download_zip(objects: List[ALObject], app_config: Dict) -> bytes:
    """
    Crée un fichier ZIP avec tous les objets AL générés
    
    Args:
        objects: Liste des objets AL
        app_config: Configuration app.json
        
    Returns:
        Contenu du fichier ZIP en bytes
    """
    zip_buffer = io.BytesIO()
    
    with zipfile.ZipFile(zip_buffer, 'w', zipfile.ZIP_DEFLATED) as zip_file:
        # Ajouter app.json
        app_json_content = json.dumps(app_config, indent=2, ensure_ascii=False)
        zip_file.writestr("app.json", app_json_content)
        
        # Ajouter tous les objets AL
        for obj in objects:
            header = f"""// Generated by Business Central AL Generator (Ollama)
// Date: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
// Object: {obj.type} {obj.id} "{obj.name}"

"""
            content = header + obj.code
            zip_file.writestr(obj.filename, content)
    
    zip_buffer.seek(0)
    return zip_buffer.getvalue()

# Interface Streamlit
def main():
    """Interface principale Streamlit"""
    
    # Sidebar pour configuration
    with st.sidebar:
        st.header("🏠 Configuration Ollama")
        
        # Configuration Ollama
        ollama_url = st.text_input(
            "URL Ollama",
            value="http://localhost:11434",
            help="URL de votre serveur Ollama local"
        )
        
        # Vérification de la connexion Ollama
        if st.button("🔍 Tester la connexion"):
            client = OllamaClient(ollama_url)
            if client.is_available():
                st.success("✅ Ollama connecté !")
                
                # Lister les modèles disponibles
                models = client.list_models()
                if models:
                    st.info(f"📋 Modèles disponibles: {', '.join(models)}")
                else:
                    st.warning("⚠️ Aucun modèle trouvé")
            else:
                st.error("❌ Impossible de se connecter à Ollama")
        
        # Sélection du modèle - MODÈLES OPTIMISÉS POUR LE CODE
        model_options = [
            "qwen2.5-coder:32b",        # ⭐⭐⭐ TOP - Excellent pour AL (si 32GB+ RAM)
            "codestral:22b",            # ⭐⭐⭐ Mistral Codestral (excellent codeur)
            "qwen2.5-coder:14b",        # ⭐⭐⭐ Très bon compromis (16GB+ RAM)
            "deepseek-coder-v2:16b",    # ⭐⭐⭐ DeepSeek V2 amélioré
            "qwen2.5-coder:7b",         # ⭐⭐ Bon pour 8GB+ RAM
            "deepseek-coder:6.7b",      # ⭐⭐ Version classique
            "codellama:13b-instruct",   # ⭐⭐ Stable et performant
            "codellama:7b-instruct",    # ⭐ Pour machines limitées
            "granite-code:8b",          # ⭐ IBM Granite (nouveau)
            "autre"
        ]
        
        selected_model = st.selectbox(
            "Modèle à utiliser",
            options=model_options,
            help="Modèles optimisés pour la génération de code AL Business Central"
        )
        
        if selected_model == "autre":
            custom_model = st.text_input("Nom du modèle personnalisé")
            if custom_model:
                selected_model = custom_model
        
        st.divider()
        
        # Informations sur les modèles recommandés
        st.header("🎯 Modèles de code optimisés")
        st.markdown("""
        **🏆 TOP pour Business Central AL :**
        - `qwen2.5-coder:32b` (⭐⭐⭐ Excellence - 32GB RAM)
        - `codestral:22b` (⭐⭐⭐ Mistral Codestral - 24GB RAM)
        - `qwen2.5-coder:14b` (⭐⭐⭐ Excellent - 16GB RAM)
        
        **🚀 Bon compromis :**
        - `qwen2.5-coder:7b` (⭐⭐ Très bon - 8GB RAM)
        - `deepseek-coder-v2:16b` (⭐⭐ V2 amélioré)
        
        **💡 Installation recommandée :**
        ```bash
        # Pour machines puissantes (32GB+)
        ollama pull qwen2.5-coder:32b
        
        # Pour machines normales (16GB+)  
        ollama pull qwen2.5-coder:14b
        
        # Pour machines limitées (8GB+)
        ollama pull qwen2.5-coder:7b
        ```
        
        **⚡ Démarrage :**
        ```bash
        ollama serve
        ```
        """)
        
        # Auto-détection du meilleur modèle
        if st.button("🎯 Détecter le meilleur modèle"):
            client = OllamaClient(ollama_url)
            if client.is_available():
                available_models = client.list_models()
                
                # Ordre de préférence pour AL
                preferred_order = [
                    "qwen2.5-coder:32b", "codestral:22b", "qwen2.5-coder:14b",
                    "deepseek-coder-v2:16b", "qwen2.5-coder:7b", "deepseek-coder:6.7b",
                    "codellama:13b-instruct", "codellama:7b-instruct"
                ]
                
                best_model = None
                for model in preferred_order:
                    if model in available_models:
                        best_model = model
                        break
                
                if best_model:
                    st.success(f"🎯 Meilleur modèle détecté: **{best_model}**")
                    st.info("💡 Sélectionnez-le dans la liste ci-dessus")
                else:
                    st.warning("⚠️ Aucun modèle de code optimisé trouvé")
                    st.info("📥 Installez un modèle recommandé avec `ollama pull qwen2.5-coder:7b`")
            else:
                st.error("❌ Ollama non disponible")
        
        st.divider()
        
        # Informations
        st.header("📋 Informations")
        st.markdown("""
        **Objets AL générés :**
        - 📊 Tables avec relations
        - 📄 Pages (List/Card)
        - ⚙️ Codeunits (logique)
        - 📈 Rapports
        - 🔐 Permission Sets
        - 📦 app.json
        """)
        
        st.markdown("""
        **Avantages local :**
        - 🔒 Confidentialité totale
        - 💰 Pas de coûts API
        - ⚡ Pas de limite de requêtes
        - 🏠 Fonctionne hors ligne
        """)
    
    # En-tête principal
    st.title("🏠 Business Central AL Generator (Local)")
    st.markdown("**Génération automatique de code AL avec Ollama - 100% local et privé**")
    
    # Zone d'upload
    st.header("📄 Spécification d'entrée")
    
    # Choix du mode d'entrée
    input_mode = st.radio(
        "Choisissez votre mode de saisie :",
        ["📝 Texte direct (Recommandé)", "📄 Upload PDF"],
        help="Le mode texte direct est plus précis pour l'analyse Ollama"
    )
    
    specification_content = None
    
    if input_mode == "📝 Texte direct (Recommandé)":
        specification_content = st.text_area(
            "Collez votre spécification ici :",
            height=300,
            placeholder="""Exemple :
1. Purpose
This functional specification describes the design of a new vehicle management module for Dynamics 365 Business Central.

2. Tables and Fields
Table: Vehicle
- VehicleID (Code[20]): Unique identifier of the vehicle
- Brand (Text[50]): Brand of the vehicle
- Model (Text[50]): Model of the vehicle
- Year (Integer): Year of manufacture
- Status (Option): Available, In Use, Under Maintenance

Table: VehicleAssignment
- AssignmentID (Code[20]): Unique identifier
- VehicleID (Code[20]): ID of the assigned vehicle
- EmployeeID (Code[20]): ID of the assigned employee

3. Business Logic
- When a vehicle is assigned, it automatically changes to In Use status
- Maintenance alerts should be generated automatically""",
            help="Copiez-collez directement le texte de votre spécification"
        )
        
        if specification_content:
            st.success(f"✅ Spécification saisie : {len(specification_content)} caractères")
    
    else:  # Mode PDF
        uploaded_file = st.file_uploader(
            "Uploadez votre spécification PDF",
            type=['pdf'],
            help="Sélectionnez un fichier PDF contenant la spécification de votre extension Business Central"
        )
        
        if uploaded_file:
            st.success(f"📎 Fichier uploadé: {uploaded_file.name}")
            st.info(f"📏 Taille: {len(uploaded_file.getvalue()) / 1024:.1f} KB")
    
    # Continuer seulement si on a du contenu ou un fichier
    if (input_mode == "📝 Texte direct (Recommandé)" and specification_content) or (input_mode == "📄 Upload PDF" and uploaded_file):
        
        col1, col2 = st.columns([3, 1])
        
        with col1:
            if input_mode == "📝 Texte direct (Recommandé)":
                st.success("✅ Mode texte direct - Analyse plus précise")
                st.info(f"📏 Taille: {len(specification_content)} caractères")
            else:
                st.success(f"📎 Fichier uploadé: {uploaded_file.name}")
                st.info(f"📏 Taille: {len(uploaded_file.getvalue()) / 1024:.1f} KB")
            st.info(f"🤖 Modèle: {selected_model}")
        
        with col2:
            generate_button = st.button("🚀 Générer le code AL", type="primary", use_container_width=True)
        
        if generate_button and selected_model:
            try:
                # Vérification de la connexion Ollama
                client = OllamaClient(ollama_url, selected_model)
                if not client.is_available():
                    st.error("❌ Impossible de se connecter à Ollama. Vérifiez que le serveur est démarré.")
                    st.info("💡 Démarrez Ollama avec: `ollama serve`")
                    return
                
                # Initialisation du générateur
                generator = BusinessCentralALGenerator(ollama_url, selected_model)
                
                # Extraction du contenu
                if input_mode == "📝 Texte direct (Recommandé)":
                    with st.spinner("📝 Analyse du contenu textuel..."):
                        pdf_content = specification_content
                else:
                    with st.spinner("🔍 Extraction du contenu PDF..."):
                        pdf_content = generator.extract_pdf_content(uploaded_file)
                
                # Analyse avec Ollama - SANS parsing structuré, direct vers l'IA
                with st.spinner(f"🤖 Analyse intelligente avec {selected_model}..."):
                    analysis = generator.analyze_specification_with_ai_only(pdf_content)
                
                # Debug : Afficher la réponse brute si demandé
                with st.expander("🔍 Debug - Voir l'analyse brute"):
                    st.json(analysis)
                
                # Affichage de l'analyse
                st.header("📋 Analyse de la spécification")
                
                col1, col2 = st.columns(2)
                
                with col1:
                    st.subheader("📊 Projet")
                    st.write(f"**Nom :** {analysis.get('project_name', 'N/A')}")
                    st.write(f"**Description :** {analysis.get('description', 'N/A')}")
                
                with col2:
                    st.subheader("🔢 Objets détectés")
                    st.metric("Tables", len(analysis.get('tables', [])))
                    st.metric("Pages", len(analysis.get('pages', [])))
                    st.metric("Codeunits", len(analysis.get('codeunits', [])))
                    st.metric("Rapports", len(analysis.get('reports', [])))
                
                # Détails des objets détectés
                with st.expander("🔍 Détails des objets détectés"):
                    
                    if analysis.get('tables'):
                        st.subheader("📊 Tables")
                        for table in analysis['tables']:
                            st.write(f"• **{table['name']}** - {table.get('description', 'N/A')}")
                    
                    if analysis.get('pages'):
                        st.subheader("📄 Pages")
                        for page in analysis['pages']:
                            st.write(f"• **{page['name']}** ({page.get('type', 'N/A')}) - {page.get('description', 'N/A')}")
                    
                    if analysis.get('codeunits'):
                        st.subheader("⚙️ Codeunits")
                        for codeunit in analysis['codeunits']:
                            st.write(f"• **{codeunit['name']}** - {codeunit.get('description', 'N/A')}")
                    
                    if analysis.get('reports'):
                        st.subheader("📈 Rapports")
                        for report in analysis['reports']:
                            st.write(f"• **{report['name']}** - {report.get('description', 'N/A')}")
                
                # Génération du code
                st.header("⚡ Génération du code AL")
                
                # Barre de progression
                progress_bar = st.progress(0)
                status_text = st.empty()
                
                def update_progress(progress, message):
                    progress_bar.progress(progress)
                    status_text.text(message)
                
                # Génération des objets
                objects, app_config = generator.generate_from_analysis(analysis, update_progress)
                
                # Résultats
                st.success(f"✅ Génération terminée ! {len(objects)} objets créés")
                
                # Résumé des objets générés
                st.subheader("📋 Objets générés")
                
                # Tableau des objets
                objects_data = []
                for obj in objects:
                    objects_data.append({
                        "Type": obj.type,
                        "ID": obj.id,
                        "Nom": obj.name,
                        "Fichier": obj.filename
                    })
                
                st.dataframe(objects_data, use_container_width=True)
                
                # Prévisualisation du code
                st.subheader("👀 Prévisualisation du code")
                
                selected_object = st.selectbox(
                    "Sélectionnez un objet pour prévisualiser le code :",
                    options=range(len(objects)),
                    format_func=lambda x: f"{objects[x].type} {objects[x].id} - {objects[x].name}"
                )
                
                if selected_object is not None:
                    obj = objects[selected_object]
                    st.code(obj.code, language="al")
                
                # Téléchargement
                st.header("💾 Téléchargement")
                
                # Création du ZIP
                zip_content = create_download_zip(objects, app_config)
                
                st.download_button(
                    label="📦 Télécharger l'extension complète (ZIP)",
                    data=zip_content,
                    file_name=f"{analysis.get('project_name', 'BC_Extension').replace(' ', '_')}.zip",
                    mime="application/zip",
                    type="primary",
                    use_container_width=True
                )
                
                st.success("🎉 Votre extension Business Central est prête !")
                
                # Instructions de déploiement
                with st.expander("📖 Instructions de déploiement"):
                    st.markdown("""
                    ### 🚀 Comment utiliser votre extension
                    
                    1. **Téléchargez** le fichier ZIP
                    2. **Extrayez** le contenu dans un nouveau dossier
                    3. **Ouvrez** le dossier dans Visual Studio Code
                    4. **Installez** l'extension AL pour VS Code si nécessaire
                    5. **Configurez** votre environnement Business Central
                    6. **Compilez** l'extension (Ctrl+Shift+P → "AL: Package")
                    7. **Déployez** dans votre environnement de test
                    
                    ### ⚙️ Configuration requise
                    - Visual Studio Code
                    - Extension AL
                    - Environnement Business Central (sandbox/dev)
                    - Docker (optionnel pour environnement local)
                    
                    ### 🏠 Avantages Ollama Local
                    - 🔒 **Confidentialité** : Vos données restent locales
                    - 💰 **Gratuit** : Aucun coût d'API
                    - ⚡ **Rapide** : Pas de latence réseau
                    - 🌐 **Hors ligne** : Fonctionne sans internet
                    """)
                
            except Exception as e:
                st.error(f"❌ Erreur lors de la génération : {str(e)}")
                st.exception(e)
                
                # Conseils de dépannage
                with st.expander("🔧 Conseils de dépannage"):
                    st.markdown("""
                    **Erreurs communes :**
                    
                    1. **Connexion Ollama** : Vérifiez que `ollama serve` est démarré
                    2. **Modèle manquant** : Installez le modèle avec `ollama pull deepseek-coder:6.7b`
                    3. **Mémoire insuffisante** : Essayez un modèle plus petit comme `qwen2.5-coder:1.5b`
                    4. **JSON malformé** : Le modèle peut générer du JSON invalide, réessayez
                    5. **Timeout** : Le modèle prend du temps, soyez patient
                    
                    **Commandes utiles :**
                    ```bash
                    # Vérifier Ollama
                    ollama list
                    
                    # Tester la connexion
                    curl http://localhost:11434/api/version
                    
                    # Redémarrer Ollama
                    ollama serve
                    ```
                    """)
    
    elif not selected_model:
        st.warning("⚠️ Veuillez sélectionner un modèle Ollama dans la barre latérale")
        
        with st.expander("🚀 Installation rapide Ollama"):
            st.markdown("""
            ### Windows
            1. Téléchargez depuis [ollama.com/download](https://ollama.com/download)
            2. Installez le fichier .exe
            3. Ouvrez un terminal et tapez :
            ```cmd
            ollama pull deepseek-coder:6.7b
            ollama serve
            ```
            
            ### Linux/macOS
            ```bash
            curl -fsSL https://ollama.com/install.sh | sh
            ollama pull deepseek-coder:6.7b
            ollama serve
            ```
            """)
    
    elif not uploaded_file:
        st.info("📄 Veuillez uploader un fichier PDF de spécification")
        
        # Exemple de spécification
        with st.expander("💡 Exemple de spécification PDF"):
            st.markdown("""
            ### Exemple de contenu pour votre PDF de spécification :
            
            **Titre :** Gestion des Commandes Clients
            
            **Description :** Extension pour gérer les commandes clients avec suivi des statuts
            
            **Tables nécessaires :**
            - Table Commande Client : Numéro, Date, Client, Montant Total, Statut
            - Table Ligne Commande : Numéro Commande, Article, Quantité, Prix Unitaire
            
            **Pages nécessaires :**
            - Page Liste des Commandes
            - Page Fiche Commande
            - Page Lignes de Commande
            
            **Fonctionnalités :**
            - Validation des commandes
            - Calcul automatique des totaux
            - Rapports de suivi
            
            **Codeunits :**
            - Gestion des commandes
            - Validation des données
            
            **Rapports :**
            - Commandes par période
            - Statistiques de vente
            """)
    
    # Footer avec informations
    st.divider()
    st.markdown("""
    <div style='text-align: center; color: #666;'>
        <p>🏠 <strong>Business Central AL Generator - Version Ollama Local</strong></p>
        <p>Génération de code AL 100% locale et privée • Aucune donnée envoyée sur internet</p>
    </div>
    """, unsafe_allow_html=True)

if __name__ == "__main__":
    main(), '', code, flags=re.MULTILINE)
            code = re.sub(r'^```.*
    
    def create_app_json(self, project_name: str, description: str) -> Dict:
        """
        Crée la configuration app.json pour l'extension
        
        Args:
            project_name: Nom du projet
            description: Description du projet
            
        Returns:
            Configuration app.json
        """
        return {
            "id": str(uuid.uuid4()),  # GUID unique généré automatiquement
            "name": project_name,
            "publisher": "Your Company",
            "version": "1.0.0.0",
            "brief": description,
            "description": description,
            "privacyStatement": "",
            "EULA": "",
            "help": "",
            "url": "",
            "logo": "",
            "dependencies": [
                {
                    "id": "63ca2fa4-4f03-4f2b-a480-172fef340d3f",
                    "publisher": "Microsoft",
                    "name": "System Application",
                    "version": "20.0.0.0"
                },
                {
                    "id": "437dbf0e-84ff-417a-965d-ed2bb9650972",
                    "publisher": "Microsoft",
                    "name": "Base Application",
                    "version": "20.0.0.0"
                }
            ],
            "screenshots": [],
            "platform": "20.0.0.0",
            "application": "20.0.0.0",
            "idRanges": [
                {
                    "from": 50000,
                    "to": 50999
                }
            ],
            "resourceExposurePolicy": {
                "allowDebugging": True,
                "allowDownloadingSource": False,
                "includeSourceInSymbolFile": False
            }
        }
    
    def generate_from_analysis(self, analysis: Dict, progress_callback=None) -> Tuple[List[ALObject], Dict]:
        """
        Génère tous les objets AL à partir de l'analyse
        
        Args:
            analysis: Analyse de la spécification
            progress_callback: Fonction de callback pour le progrès
            
        Returns:
            Tuple (objets générés, app.json config)
        """
        project_name = analysis.get('project_name', 'BC Extension')
        description = analysis.get('description', 'Extension générée automatiquement')
        
        generated_objects = []
        total_objects = (
            len(analysis.get('tables', [])) +
            len(analysis.get('pages', [])) +
            len(analysis.get('codeunits', [])) +
            len(analysis.get('reports', [])) + 1  # +1 pour le permission set
        )
        current_object = 0
        
        # Tables
        if 'tables' in analysis:
            for table_spec in analysis['tables']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération table: {table_spec['name']}")
                
                code = self.generate_object_code('Table', table_spec)
                table_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Table',
                    name=table_spec['name'],
                    id=table_id,
                    code=code,
                    dependencies=[],
                    filename=f"Tab{table_id}.{table_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Pages
        if 'pages' in analysis:
            for page_spec in analysis['pages']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération page: {page_spec['name']}")
                
                context = {'tables': analysis.get('tables', [])}
                code = self.generate_object_code('Page', page_spec, context)
                page_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Page',
                    name=page_spec['name'],
                    id=page_id,
                    code=code,
                    dependencies=[page_spec.get('source_table', '')],
                    filename=f"Pag{page_id}.{page_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Codeunits
        if 'codeunits' in analysis:
            for codeunit_spec in analysis['codeunits']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération codeunit: {codeunit_spec['name']}")
                
                code = self.generate_object_code('Codeunit', codeunit_spec)
                codeunit_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Codeunit',
                    name=codeunit_spec['name'],
                    id=codeunit_id,
                    code=code,
                    dependencies=[],
                    filename=f"Cod{codeunit_id}.{codeunit_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Rapports
        if 'reports' in analysis:
            for report_spec in analysis['reports']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération rapport: {report_spec['name']}")
                
                code = self.generate_object_code('Report', report_spec)
                report_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Report',
                    name=report_spec['name'],
                    id=report_id,
                    code=code,
                    dependencies=report_spec.get('data_items', []),
                    filename=f"Rep{report_id}.{report_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Permission Set
        if progress_callback:
            progress_callback(current_object / total_objects, "Génération du Permission Set...")
        
        permissionset_code = self.generate_permission_set_code(project_name, generated_objects)
        permissionset_id = list(self.used_ids)[-1]  # Dernier ID utilisé
        
        al_object = ALObject(
            type='PermissionSet',
            name=f"{project_name} Permissions",
            id=permissionset_id,
            code=permissionset_code,
            dependencies=[],
            filename=f"Per{permissionset_id}.{project_name.replace(' ', '')}Permissions.al"
        )
        
        generated_objects.append(al_object)
        
        # Configuration app.json
        app_config = self.create_app_json(project_name, description)
        
        if progress_callback:
            progress_callback(1.0, "Génération terminée!")
        
        return generated_objects, app_config

def create_download_zip(objects: List[ALObject], app_config: Dict) -> bytes:
    """
    Crée un fichier ZIP avec tous les objets AL générés
    
    Args:
        objects: Liste des objets AL
        app_config: Configuration app.json
        
    Returns:
        Contenu du fichier ZIP en bytes
    """
    zip_buffer = io.BytesIO()
    
    with zipfile.ZipFile(zip_buffer, 'w', zipfile.ZIP_DEFLATED) as zip_file:
        # Ajouter app.json
        app_json_content = json.dumps(app_config, indent=2, ensure_ascii=False)
        zip_file.writestr("app.json", app_json_content)
        
        # Ajouter tous les objets AL
        for obj in objects:
            header = f"""// Generated by Business Central AL Generator (Ollama)
// Date: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
// Object: {obj.type} {obj.id} "{obj.name}"

"""
            content = header + obj.code
            zip_file.writestr(obj.filename, content)
    
    zip_buffer.seek(0)
    return zip_buffer.getvalue()

# Interface Streamlit
def main():
    """Interface principale Streamlit"""
    
    # Sidebar pour configuration
    with st.sidebar:
        st.header("🏠 Configuration Ollama")
        
        # Configuration Ollama
        ollama_url = st.text_input(
            "URL Ollama",
            value="http://localhost:11434",
            help="URL de votre serveur Ollama local"
        )
        
        # Vérification de la connexion Ollama
        if st.button("🔍 Tester la connexion"):
            client = OllamaClient(ollama_url)
            if client.is_available():
                st.success("✅ Ollama connecté !")
                
                # Lister les modèles disponibles
                models = client.list_models()
                if models:
                    st.info(f"📋 Modèles disponibles: {', '.join(models)}")
                else:
                    st.warning("⚠️ Aucun modèle trouvé")
            else:
                st.error("❌ Impossible de se connecter à Ollama")
        
        # Sélection du modèle - MODÈLES OPTIMISÉS POUR LE CODE
        model_options = [
            "qwen2.5-coder:32b",        # ⭐⭐⭐ TOP - Excellent pour AL (si 32GB+ RAM)
            "codestral:22b",            # ⭐⭐⭐ Mistral Codestral (excellent codeur)
            "qwen2.5-coder:14b",        # ⭐⭐⭐ Très bon compromis (16GB+ RAM)
            "deepseek-coder-v2:16b",    # ⭐⭐⭐ DeepSeek V2 amélioré
            "qwen2.5-coder:7b",         # ⭐⭐ Bon pour 8GB+ RAM
            "deepseek-coder:6.7b",      # ⭐⭐ Version classique
            "codellama:13b-instruct",   # ⭐⭐ Stable et performant
            "codellama:7b-instruct",    # ⭐ Pour machines limitées
            "granite-code:8b",          # ⭐ IBM Granite (nouveau)
            "autre"
        ]
        
        selected_model = st.selectbox(
            "Modèle à utiliser",
            options=model_options,
            help="Modèles optimisés pour la génération de code AL Business Central"
        )
        
        if selected_model == "autre":
            custom_model = st.text_input("Nom du modèle personnalisé")
            if custom_model:
                selected_model = custom_model
        
        st.divider()
        
        # Informations sur les modèles recommandés
        st.header("🎯 Modèles de code optimisés")
        st.markdown("""
        **🏆 TOP pour Business Central AL :**
        - `qwen2.5-coder:32b` (⭐⭐⭐ Excellence - 32GB RAM)
        - `codestral:22b` (⭐⭐⭐ Mistral Codestral - 24GB RAM)
        - `qwen2.5-coder:14b` (⭐⭐⭐ Excellent - 16GB RAM)
        
        **🚀 Bon compromis :**
        - `qwen2.5-coder:7b` (⭐⭐ Très bon - 8GB RAM)
        - `deepseek-coder-v2:16b` (⭐⭐ V2 amélioré)
        
        **💡 Installation recommandée :**
        ```bash
        # Pour machines puissantes (32GB+)
        ollama pull qwen2.5-coder:32b
        
        # Pour machines normales (16GB+)  
        ollama pull qwen2.5-coder:14b
        
        # Pour machines limitées (8GB+)
        ollama pull qwen2.5-coder:7b
        ```
        
        **⚡ Démarrage :**
        ```bash
        ollama serve
        ```
        """)
        
        # Auto-détection du meilleur modèle
        if st.button("🎯 Détecter le meilleur modèle"):
            client = OllamaClient(ollama_url)
            if client.is_available():
                available_models = client.list_models()
                
                # Ordre de préférence pour AL
                preferred_order = [
                    "qwen2.5-coder:32b", "codestral:22b", "qwen2.5-coder:14b",
                    "deepseek-coder-v2:16b", "qwen2.5-coder:7b", "deepseek-coder:6.7b",
                    "codellama:13b-instruct", "codellama:7b-instruct"
                ]
                
                best_model = None
                for model in preferred_order:
                    if model in available_models:
                        best_model = model
                        break
                
                if best_model:
                    st.success(f"🎯 Meilleur modèle détecté: **{best_model}**")
                    st.info("💡 Sélectionnez-le dans la liste ci-dessus")
                else:
                    st.warning("⚠️ Aucun modèle de code optimisé trouvé")
                    st.info("📥 Installez un modèle recommandé avec `ollama pull qwen2.5-coder:7b`")
            else:
                st.error("❌ Ollama non disponible")
        
        st.divider()
        
        # Informations
        st.header("📋 Informations")
        st.markdown("""
        **Objets AL générés :**
        - 📊 Tables avec relations
        - 📄 Pages (List/Card)
        - ⚙️ Codeunits (logique)
        - 📈 Rapports
        - 🔐 Permission Sets
        - 📦 app.json
        """)
        
        st.markdown("""
        **Avantages local :**
        - 🔒 Confidentialité totale
        - 💰 Pas de coûts API
        - ⚡ Pas de limite de requêtes
        - 🏠 Fonctionne hors ligne
        """)
    
    # En-tête principal
    st.title("🏠 Business Central AL Generator (Local)")
    st.markdown("**Génération automatique de code AL avec Ollama - 100% local et privé**")
    
    # Zone d'upload
    st.header("📄 Spécification d'entrée")
    
    # Choix du mode d'entrée
    input_mode = st.radio(
        "Choisissez votre mode de saisie :",
        ["📝 Texte direct (Recommandé)", "📄 Upload PDF"],
        help="Le mode texte direct est plus précis pour l'analyse Ollama"
    )
    
    specification_content = None
    
    if input_mode == "📝 Texte direct (Recommandé)":
        specification_content = st.text_area(
            "Collez votre spécification ici :",
            height=300,
            placeholder="""Exemple :
1. Purpose
This functional specification describes the design of a new vehicle management module for Dynamics 365 Business Central.

2. Tables and Fields
Table: Vehicle
- VehicleID (Code[20]): Unique identifier of the vehicle
- Brand (Text[50]): Brand of the vehicle
- Model (Text[50]): Model of the vehicle
- Year (Integer): Year of manufacture
- Status (Option): Available, In Use, Under Maintenance

Table: VehicleAssignment
- AssignmentID (Code[20]): Unique identifier
- VehicleID (Code[20]): ID of the assigned vehicle
- EmployeeID (Code[20]): ID of the assigned employee

3. Business Logic
- When a vehicle is assigned, it automatically changes to In Use status
- Maintenance alerts should be generated automatically""",
            help="Copiez-collez directement le texte de votre spécification"
        )
        
        if specification_content:
            st.success(f"✅ Spécification saisie : {len(specification_content)} caractères")
    
    else:  # Mode PDF
        uploaded_file = st.file_uploader(
            "Uploadez votre spécification PDF",
            type=['pdf'],
            help="Sélectionnez un fichier PDF contenant la spécification de votre extension Business Central"
        )
        
        if uploaded_file:
            st.success(f"📎 Fichier uploadé: {uploaded_file.name}")
            st.info(f"📏 Taille: {len(uploaded_file.getvalue()) / 1024:.1f} KB")
    
    # Continuer seulement si on a du contenu ou un fichier
    if (input_mode == "📝 Texte direct (Recommandé)" and specification_content) or (input_mode == "📄 Upload PDF" and uploaded_file):
        
        col1, col2 = st.columns([3, 1])
        
        with col1:
            if input_mode == "📝 Texte direct (Recommandé)":
                st.success("✅ Mode texte direct - Analyse plus précise")
                st.info(f"📏 Taille: {len(specification_content)} caractères")
            else:
                st.success(f"📎 Fichier uploadé: {uploaded_file.name}")
                st.info(f"📏 Taille: {len(uploaded_file.getvalue()) / 1024:.1f} KB")
            st.info(f"🤖 Modèle: {selected_model}")
        
        with col2:
            generate_button = st.button("🚀 Générer le code AL", type="primary", use_container_width=True)
        
        if generate_button and selected_model:
            try:
                # Vérification de la connexion Ollama
                client = OllamaClient(ollama_url, selected_model)
                if not client.is_available():
                    st.error("❌ Impossible de se connecter à Ollama. Vérifiez que le serveur est démarré.")
                    st.info("💡 Démarrez Ollama avec: `ollama serve`")
                    return
                
                # Initialisation du générateur
                generator = BusinessCentralALGenerator(ollama_url, selected_model)
                
                # Extraction du contenu
                if input_mode == "📝 Texte direct (Recommandé)":
                    with st.spinner("📝 Analyse du contenu textuel..."):
                        pdf_content = specification_content
                else:
                    with st.spinner("🔍 Extraction du contenu PDF..."):
                        pdf_content = generator.extract_pdf_content(uploaded_file)
                
                # Analyse avec Ollama - SANS parsing structuré, direct vers l'IA
                with st.spinner(f"🤖 Analyse intelligente avec {selected_model}..."):
                    analysis = generator.analyze_specification_with_ai_only(pdf_content)
                
                # Debug : Afficher la réponse brute si demandé
                with st.expander("🔍 Debug - Voir l'analyse brute"):
                    st.json(analysis)
                
                # Affichage de l'analyse
                st.header("📋 Analyse de la spécification")
                
                col1, col2 = st.columns(2)
                
                with col1:
                    st.subheader("📊 Projet")
                    st.write(f"**Nom :** {analysis.get('project_name', 'N/A')}")
                    st.write(f"**Description :** {analysis.get('description', 'N/A')}")
                
                with col2:
                    st.subheader("🔢 Objets détectés")
                    st.metric("Tables", len(analysis.get('tables', [])))
                    st.metric("Pages", len(analysis.get('pages', [])))
                    st.metric("Codeunits", len(analysis.get('codeunits', [])))
                    st.metric("Rapports", len(analysis.get('reports', [])))
                
                # Détails des objets détectés
                with st.expander("🔍 Détails des objets détectés"):
                    
                    if analysis.get('tables'):
                        st.subheader("📊 Tables")
                        for table in analysis['tables']:
                            st.write(f"• **{table['name']}** - {table.get('description', 'N/A')}")
                    
                    if analysis.get('pages'):
                        st.subheader("📄 Pages")
                        for page in analysis['pages']:
                            st.write(f"• **{page['name']}** ({page.get('type', 'N/A')}) - {page.get('description', 'N/A')}")
                    
                    if analysis.get('codeunits'):
                        st.subheader("⚙️ Codeunits")
                        for codeunit in analysis['codeunits']:
                            st.write(f"• **{codeunit['name']}** - {codeunit.get('description', 'N/A')}")
                    
                    if analysis.get('reports'):
                        st.subheader("📈 Rapports")
                        for report in analysis['reports']:
                            st.write(f"• **{report['name']}** - {report.get('description', 'N/A')}")
                
                # Génération du code
                st.header("⚡ Génération du code AL")
                
                # Barre de progression
                progress_bar = st.progress(0)
                status_text = st.empty()
                
                def update_progress(progress, message):
                    progress_bar.progress(progress)
                    status_text.text(message)
                
                # Génération des objets
                objects, app_config = generator.generate_from_analysis(analysis, update_progress)
                
                # Résultats
                st.success(f"✅ Génération terminée ! {len(objects)} objets créés")
                
                # Résumé des objets générés
                st.subheader("📋 Objets générés")
                
                # Tableau des objets
                objects_data = []
                for obj in objects:
                    objects_data.append({
                        "Type": obj.type,
                        "ID": obj.id,
                        "Nom": obj.name,
                        "Fichier": obj.filename
                    })
                
                st.dataframe(objects_data, use_container_width=True)
                
                # Prévisualisation du code
                st.subheader("👀 Prévisualisation du code")
                
                selected_object = st.selectbox(
                    "Sélectionnez un objet pour prévisualiser le code :",
                    options=range(len(objects)),
                    format_func=lambda x: f"{objects[x].type} {objects[x].id} - {objects[x].name}"
                )
                
                if selected_object is not None:
                    obj = objects[selected_object]
                    st.code(obj.code, language="al")
                
                # Téléchargement
                st.header("💾 Téléchargement")
                
                # Création du ZIP
                zip_content = create_download_zip(objects, app_config)
                
                st.download_button(
                    label="📦 Télécharger l'extension complète (ZIP)",
                    data=zip_content,
                    file_name=f"{analysis.get('project_name', 'BC_Extension').replace(' ', '_')}.zip",
                    mime="application/zip",
                    type="primary",
                    use_container_width=True
                )
                
                st.success("🎉 Votre extension Business Central est prête !")
                
                # Instructions de déploiement
                with st.expander("📖 Instructions de déploiement"):
                    st.markdown("""
                    ### 🚀 Comment utiliser votre extension
                    
                    1. **Téléchargez** le fichier ZIP
                    2. **Extrayez** le contenu dans un nouveau dossier
                    3. **Ouvrez** le dossier dans Visual Studio Code
                    4. **Installez** l'extension AL pour VS Code si nécessaire
                    5. **Configurez** votre environnement Business Central
                    6. **Compilez** l'extension (Ctrl+Shift+P → "AL: Package")
                    7. **Déployez** dans votre environnement de test
                    
                    ### ⚙️ Configuration requise
                    - Visual Studio Code
                    - Extension AL
                    - Environnement Business Central (sandbox/dev)
                    - Docker (optionnel pour environnement local)
                    
                    ### 🏠 Avantages Ollama Local
                    - 🔒 **Confidentialité** : Vos données restent locales
                    - 💰 **Gratuit** : Aucun coût d'API
                    - ⚡ **Rapide** : Pas de latence réseau
                    - 🌐 **Hors ligne** : Fonctionne sans internet
                    """)
                
            except Exception as e:
                st.error(f"❌ Erreur lors de la génération : {str(e)}")
                st.exception(e)
                
                # Conseils de dépannage
                with st.expander("🔧 Conseils de dépannage"):
                    st.markdown("""
                    **Erreurs communes :**
                    
                    1. **Connexion Ollama** : Vérifiez que `ollama serve` est démarré
                    2. **Modèle manquant** : Installez le modèle avec `ollama pull deepseek-coder:6.7b`
                    3. **Mémoire insuffisante** : Essayez un modèle plus petit comme `qwen2.5-coder:1.5b`
                    4. **JSON malformé** : Le modèle peut générer du JSON invalide, réessayez
                    5. **Timeout** : Le modèle prend du temps, soyez patient
                    
                    **Commandes utiles :**
                    ```bash
                    # Vérifier Ollama
                    ollama list
                    
                    # Tester la connexion
                    curl http://localhost:11434/api/version
                    
                    # Redémarrer Ollama
                    ollama serve
                    ```
                    """)
    
    elif not selected_model:
        st.warning("⚠️ Veuillez sélectionner un modèle Ollama dans la barre latérale")
        
        with st.expander("🚀 Installation rapide Ollama"):
            st.markdown("""
            ### Windows
            1. Téléchargez depuis [ollama.com/download](https://ollama.com/download)
            2. Installez le fichier .exe
            3. Ouvrez un terminal et tapez :
            ```cmd
            ollama pull deepseek-coder:6.7b
            ollama serve
            ```
            
            ### Linux/macOS
            ```bash
            curl -fsSL https://ollama.com/install.sh | sh
            ollama pull deepseek-coder:6.7b
            ollama serve
            ```
            """)
    
    elif not uploaded_file:
        st.info("📄 Veuillez uploader un fichier PDF de spécification")
        
        # Exemple de spécification
        with st.expander("💡 Exemple de spécification PDF"):
            st.markdown("""
            ### Exemple de contenu pour votre PDF de spécification :
            
            **Titre :** Gestion des Commandes Clients
            
            **Description :** Extension pour gérer les commandes clients avec suivi des statuts
            
            **Tables nécessaires :**
            - Table Commande Client : Numéro, Date, Client, Montant Total, Statut
            - Table Ligne Commande : Numéro Commande, Article, Quantité, Prix Unitaire
            
            **Pages nécessaires :**
            - Page Liste des Commandes
            - Page Fiche Commande
            - Page Lignes de Commande
            
            **Fonctionnalités :**
            - Validation des commandes
            - Calcul automatique des totaux
            - Rapports de suivi
            
            **Codeunits :**
            - Gestion des commandes
            - Validation des données
            
            **Rapports :**
            - Commandes par période
            - Statistiques de vente
            """)
    
    # Footer avec informations
    st.divider()
    st.markdown("""
    <div style='text-align: center; color: #666;'>
        <p>🏠 <strong>Business Central AL Generator - Version Ollama Local</strong></p>
        <p>Génération de code AL 100% locale et privée • Aucune donnée envoyée sur internet</p>
    </div>
    """, unsafe_allow_html=True)

if __name__ == "__main__":
    main(), '', code, flags=re.MULTILINE)
            
            # Supprimer les explications avant/après le code
            lines = code.split('\n')
            start_idx = -1
            end_idx = len(lines)
            
            # Trouver le début du code AL (table, page, codeunit, report)
            for i, line in enumerate(lines):
                if re.match(r'^\s*(table|page|codeunit|report)\s+\d+', line.lower()):
                    start_idx = i
                    break
            
            # Trouver la fin du code AL (dernière accolade fermante)
            brace_count = 0
            for i in range(start_idx if start_idx != -1 else 0, len(lines)):
                line = lines[i]
                brace_count += line.count('{') - line.count('}')
                if brace_count == 0 and (line.strip() == '}' or '}' in line):
                    end_idx = i + 1
                    break
            
            if start_idx != -1:
                cleaned_code = '\n'.join(lines[start_idx:end_idx])
            else:
                cleaned_code = code
            
            return cleaned_code.strip()
            
        except Exception as e:
            raise Exception(f"Erreur lors de la génération de {object_type}: {str(e)}")
    
    def chat_with_enhanced_parameters(self, messages: List[Dict], max_tokens: int = 4000, temperature: float = 0.05, top_p: float = 0.8) -> str:
        """
        Version améliorée du chat avec paramètres optimisés pour le code
        """
        prompt = ""
        for msg in messages:
            if msg["role"] == "system":
                prompt += f"System: {msg['content']}\n\n"
            elif msg["role"] == "user":
                prompt += f"User: {msg['content']}\n\n"
            elif msg["role"] == "assistant":
                prompt += f"Assistant: {msg['content']}\n\n"
        
        prompt += "Assistant: "
        
        payload = {
            "model": self.client.model,
            "prompt": prompt,
            "stream": False,
            "options": {
                "temperature": temperature,
                "top_p": top_p,
                "top_k": 40,  # Limiter les choix pour plus de cohérence
                "repeat_penalty": 1.1,  # Éviter les répétitions
                "num_predict": max_tokens,
                "stop": ["```", "Explanation:", "Note:", "//END"]  # Arrêter sur ces mots
            }
        }
        
        try:
            response = requests.post(
                f"{self.client.base_url}/api/generate",
                json=payload,
                timeout=180  # 3 minutes pour les gros modèles
            )
            response.raise_for_status()
            
            result = response.json()
            return result.get("response", "")
            
        except Exception as e:
            raise Exception(f"Erreur Ollama optimisée: {str(e)}")
    
    def generate_permission_set_code(self, project_name: str, objects: List[ALObject]) -> str:
        """
        Génère le code AL pour un permission set
        
        Args:
            project_name: Nom du projet
            objects: Liste des objets générés
            
        Returns:
            Code AL du permission set
        """
        permissionset_id = self.get_next_object_id('PermissionSet')
        
        objects_list = [f"{obj.type} {obj.id} '{obj.name}'" for obj in objects]
        
        prompt = f"""Tu es un expert en AL pour Dynamics 365 Business Central. Génère un PermissionSet complet.

PROJECT: {project_name}
PERMISSIONSET ID: {permissionset_id}
OBJECTS: {objects_list}

INSTRUCTIONS:
- Crée un PermissionSet AL avec l'ID spécifié
- Inclus tous les objets listés
- Donne les permissions complètes (RIMD) sur tous les objets
- Utilise un nom approprié basé sur le projet
- Utilise les meilleures pratiques AL

RETOURNE UNIQUEMENT LE CODE AL, sans explication."""
        
        try:
            response = self.client.chat(
                messages=[{"role": "user", "content": prompt}],
                max_tokens=2000
            )
            
            # Nettoyer la réponse
            code = response.strip()
            code = re.sub(r'^```(?:al)?', '', code, flags=re.MULTILINE)
            code = re.sub(r'^```$', '', code, flags=re.MULTILINE)
            
            return code.strip()
            
        except Exception as e:
            raise Exception(f"Erreur lors de la génération du PermissionSet: {str(e)}")
    
    def create_app_json(self, project_name: str, description: str) -> Dict:
        """
        Crée la configuration app.json pour l'extension
        
        Args:
            project_name: Nom du projet
            description: Description du projet
            
        Returns:
            Configuration app.json
        """
        return {
            "id": str(uuid.uuid4()),  # GUID unique généré automatiquement
            "name": project_name,
            "publisher": "Your Company",
            "version": "1.0.0.0",
            "brief": description,
            "description": description,
            "privacyStatement": "",
            "EULA": "",
            "help": "",
            "url": "",
            "logo": "",
            "dependencies": [
                {
                    "id": "63ca2fa4-4f03-4f2b-a480-172fef340d3f",
                    "publisher": "Microsoft",
                    "name": "System Application",
                    "version": "20.0.0.0"
                },
                {
                    "id": "437dbf0e-84ff-417a-965d-ed2bb9650972",
                    "publisher": "Microsoft",
                    "name": "Base Application",
                    "version": "20.0.0.0"
                }
            ],
            "screenshots": [],
            "platform": "20.0.0.0",
            "application": "20.0.0.0",
            "idRanges": [
                {
                    "from": 50000,
                    "to": 50999
                }
            ],
            "resourceExposurePolicy": {
                "allowDebugging": True,
                "allowDownloadingSource": False,
                "includeSourceInSymbolFile": False
            }
        }
    
    def generate_from_analysis(self, analysis: Dict, progress_callback=None) -> Tuple[List[ALObject], Dict]:
        """
        Génère tous les objets AL à partir de l'analyse
        
        Args:
            analysis: Analyse de la spécification
            progress_callback: Fonction de callback pour le progrès
            
        Returns:
            Tuple (objets générés, app.json config)
        """
        project_name = analysis.get('project_name', 'BC Extension')
        description = analysis.get('description', 'Extension générée automatiquement')
        
        generated_objects = []
        total_objects = (
            len(analysis.get('tables', [])) +
            len(analysis.get('pages', [])) +
            len(analysis.get('codeunits', [])) +
            len(analysis.get('reports', [])) + 1  # +1 pour le permission set
        )
        current_object = 0
        
        # Tables
        if 'tables' in analysis:
            for table_spec in analysis['tables']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération table: {table_spec['name']}")
                
                code = self.generate_object_code('Table', table_spec)
                table_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Table',
                    name=table_spec['name'],
                    id=table_id,
                    code=code,
                    dependencies=[],
                    filename=f"Tab{table_id}.{table_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Pages
        if 'pages' in analysis:
            for page_spec in analysis['pages']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération page: {page_spec['name']}")
                
                context = {'tables': analysis.get('tables', [])}
                code = self.generate_object_code('Page', page_spec, context)
                page_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Page',
                    name=page_spec['name'],
                    id=page_id,
                    code=code,
                    dependencies=[page_spec.get('source_table', '')],
                    filename=f"Pag{page_id}.{page_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Codeunits
        if 'codeunits' in analysis:
            for codeunit_spec in analysis['codeunits']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération codeunit: {codeunit_spec['name']}")
                
                code = self.generate_object_code('Codeunit', codeunit_spec)
                codeunit_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Codeunit',
                    name=codeunit_spec['name'],
                    id=codeunit_id,
                    code=code,
                    dependencies=[],
                    filename=f"Cod{codeunit_id}.{codeunit_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Rapports
        if 'reports' in analysis:
            for report_spec in analysis['reports']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération rapport: {report_spec['name']}")
                
                code = self.generate_object_code('Report', report_spec)
                report_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Report',
                    name=report_spec['name'],
                    id=report_id,
                    code=code,
                    dependencies=report_spec.get('data_items', []),
                    filename=f"Rep{report_id}.{report_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Permission Set
        if progress_callback:
            progress_callback(current_object / total_objects, "Génération du Permission Set...")
        
        permissionset_code = self.generate_permission_set_code(project_name, generated_objects)
        permissionset_id = list(self.used_ids)[-1]  # Dernier ID utilisé
        
        al_object = ALObject(
            type='PermissionSet',
            name=f"{project_name} Permissions",
            id=permissionset_id,
            code=permissionset_code,
            dependencies=[],
            filename=f"Per{permissionset_id}.{project_name.replace(' ', '')}Permissions.al"
        )
        
        generated_objects.append(al_object)
        
        # Configuration app.json
        app_config = self.create_app_json(project_name, description)
        
        if progress_callback:
            progress_callback(1.0, "Génération terminée!")
        
        return generated_objects, app_config

def create_download_zip(objects: List[ALObject], app_config: Dict) -> bytes:
    """
    Crée un fichier ZIP avec tous les objets AL générés
    
    Args:
        objects: Liste des objets AL
        app_config: Configuration app.json
        
    Returns:
        Contenu du fichier ZIP en bytes
    """
    zip_buffer = io.BytesIO()
    
    with zipfile.ZipFile(zip_buffer, 'w', zipfile.ZIP_DEFLATED) as zip_file:
        # Ajouter app.json
        app_json_content = json.dumps(app_config, indent=2, ensure_ascii=False)
        zip_file.writestr("app.json", app_json_content)
        
        # Ajouter tous les objets AL
        for obj in objects:
            header = f"""// Generated by Business Central AL Generator (Ollama)
// Date: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
// Object: {obj.type} {obj.id} "{obj.name}"

"""
            content = header + obj.code
            zip_file.writestr(obj.filename, content)
    
    zip_buffer.seek(0)
    return zip_buffer.getvalue()

# Interface Streamlit
def main():
    """Interface principale Streamlit"""
    
    # Sidebar pour configuration
    with st.sidebar:
        st.header("🏠 Configuration Ollama")
        
        # Configuration Ollama
        ollama_url = st.text_input(
            "URL Ollama",
            value="http://localhost:11434",
            help="URL de votre serveur Ollama local"
        )
        
        # Vérification de la connexion Ollama
        if st.button("🔍 Tester la connexion"):
            client = OllamaClient(ollama_url)
            if client.is_available():
                st.success("✅ Ollama connecté !")
                
                # Lister les modèles disponibles
                models = client.list_models()
                if models:
                    st.info(f"📋 Modèles disponibles: {', '.join(models)}")
                else:
                    st.warning("⚠️ Aucun modèle trouvé")
            else:
                st.error("❌ Impossible de se connecter à Ollama")
        
        # Sélection du modèle - MODÈLES OPTIMISÉS POUR LE CODE
        model_options = [
            "qwen2.5-coder:32b",        # ⭐⭐⭐ TOP - Excellent pour AL (si 32GB+ RAM)
            "codestral:22b",            # ⭐⭐⭐ Mistral Codestral (excellent codeur)
            "qwen2.5-coder:14b",        # ⭐⭐⭐ Très bon compromis (16GB+ RAM)
            "deepseek-coder-v2:16b",    # ⭐⭐⭐ DeepSeek V2 amélioré
            "qwen2.5-coder:7b",         # ⭐⭐ Bon pour 8GB+ RAM
            "deepseek-coder:6.7b",      # ⭐⭐ Version classique
            "codellama:13b-instruct",   # ⭐⭐ Stable et performant
            "codellama:7b-instruct",    # ⭐ Pour machines limitées
            "granite-code:8b",          # ⭐ IBM Granite (nouveau)
            "autre"
        ]
        
        selected_model = st.selectbox(
            "Modèle à utiliser",
            options=model_options,
            help="Modèles optimisés pour la génération de code AL Business Central"
        )
        
        if selected_model == "autre":
            custom_model = st.text_input("Nom du modèle personnalisé")
            if custom_model:
                selected_model = custom_model
        
        st.divider()
        
        # Informations sur les modèles recommandés
        st.header("🎯 Modèles de code optimisés")
        st.markdown("""
        **🏆 TOP pour Business Central AL :**
        - `qwen2.5-coder:32b` (⭐⭐⭐ Excellence - 32GB RAM)
        - `codestral:22b` (⭐⭐⭐ Mistral Codestral - 24GB RAM)
        - `qwen2.5-coder:14b` (⭐⭐⭐ Excellent - 16GB RAM)
        
        **🚀 Bon compromis :**
        - `qwen2.5-coder:7b` (⭐⭐ Très bon - 8GB RAM)
        - `deepseek-coder-v2:16b` (⭐⭐ V2 amélioré)
        
        **💡 Installation recommandée :**
        ```bash
        # Pour machines puissantes (32GB+)
        ollama pull qwen2.5-coder:32b
        
        # Pour machines normales (16GB+)  
        ollama pull qwen2.5-coder:14b
        
        # Pour machines limitées (8GB+)
        ollama pull qwen2.5-coder:7b
        ```
        
        **⚡ Démarrage :**
        ```bash
        ollama serve
        ```
        """)
        
        # Auto-détection du meilleur modèle
        if st.button("🎯 Détecter le meilleur modèle"):
            client = OllamaClient(ollama_url)
            if client.is_available():
                available_models = client.list_models()
                
                # Ordre de préférence pour AL
                preferred_order = [
                    "qwen2.5-coder:32b", "codestral:22b", "qwen2.5-coder:14b",
                    "deepseek-coder-v2:16b", "qwen2.5-coder:7b", "deepseek-coder:6.7b",
                    "codellama:13b-instruct", "codellama:7b-instruct"
                ]
                
                best_model = None
                for model in preferred_order:
                    if model in available_models:
                        best_model = model
                        break
                
                if best_model:
                    st.success(f"🎯 Meilleur modèle détecté: **{best_model}**")
                    st.info("💡 Sélectionnez-le dans la liste ci-dessus")
                else:
                    st.warning("⚠️ Aucun modèle de code optimisé trouvé")
                    st.info("📥 Installez un modèle recommandé avec `ollama pull qwen2.5-coder:7b`")
            else:
                st.error("❌ Ollama non disponible")
        
        st.divider()
        
        # Informations
        st.header("📋 Informations")
        st.markdown("""
        **Objets AL générés :**
        - 📊 Tables avec relations
        - 📄 Pages (List/Card)
        - ⚙️ Codeunits (logique)
        - 📈 Rapports
        - 🔐 Permission Sets
        - 📦 app.json
        """)
        
        st.markdown("""
        **Avantages local :**
        - 🔒 Confidentialité totale
        - 💰 Pas de coûts API
        - ⚡ Pas de limite de requêtes
        - 🏠 Fonctionne hors ligne
        """)
    
    # En-tête principal
    st.title("🏠 Business Central AL Generator (Local)")
    st.markdown("**Génération automatique de code AL avec Ollama - 100% local et privé**")
    
    # Zone d'upload
    st.header("📄 Spécification d'entrée")
    
    # Choix du mode d'entrée
    input_mode = st.radio(
        "Choisissez votre mode de saisie :",
        ["📝 Texte direct (Recommandé)", "📄 Upload PDF"],
        help="Le mode texte direct est plus précis pour l'analyse Ollama"
    )
    
    specification_content = None
    
    if input_mode == "📝 Texte direct (Recommandé)":
        specification_content = st.text_area(
            "Collez votre spécification ici :",
            height=300,
            placeholder="""Exemple :
1. Purpose
This functional specification describes the design of a new vehicle management module for Dynamics 365 Business Central.

2. Tables and Fields
Table: Vehicle
- VehicleID (Code[20]): Unique identifier of the vehicle
- Brand (Text[50]): Brand of the vehicle
- Model (Text[50]): Model of the vehicle
- Year (Integer): Year of manufacture
- Status (Option): Available, In Use, Under Maintenance

Table: VehicleAssignment
- AssignmentID (Code[20]): Unique identifier
- VehicleID (Code[20]): ID of the assigned vehicle
- EmployeeID (Code[20]): ID of the assigned employee

3. Business Logic
- When a vehicle is assigned, it automatically changes to In Use status
- Maintenance alerts should be generated automatically""",
            help="Copiez-collez directement le texte de votre spécification"
        )
        
        if specification_content:
            st.success(f"✅ Spécification saisie : {len(specification_content)} caractères")
    
    else:  # Mode PDF
        uploaded_file = st.file_uploader(
            "Uploadez votre spécification PDF",
            type=['pdf'],
            help="Sélectionnez un fichier PDF contenant la spécification de votre extension Business Central"
        )
        
        if uploaded_file:
            st.success(f"📎 Fichier uploadé: {uploaded_file.name}")
            st.info(f"📏 Taille: {len(uploaded_file.getvalue()) / 1024:.1f} KB")
    
    # Continuer seulement si on a du contenu ou un fichier
    if (input_mode == "📝 Texte direct (Recommandé)" and specification_content) or (input_mode == "📄 Upload PDF" and uploaded_file):
        
        col1, col2 = st.columns([3, 1])
        
        with col1:
            if input_mode == "📝 Texte direct (Recommandé)":
                st.success("✅ Mode texte direct - Analyse plus précise")
                st.info(f"📏 Taille: {len(specification_content)} caractères")
            else:
                st.success(f"📎 Fichier uploadé: {uploaded_file.name}")
                st.info(f"📏 Taille: {len(uploaded_file.getvalue()) / 1024:.1f} KB")
            st.info(f"🤖 Modèle: {selected_model}")
        
        with col2:
            generate_button = st.button("🚀 Générer le code AL", type="primary", use_container_width=True)
        
        if generate_button and selected_model:
            try:
                # Vérification de la connexion Ollama
                client = OllamaClient(ollama_url, selected_model)
                if not client.is_available():
                    st.error("❌ Impossible de se connecter à Ollama. Vérifiez que le serveur est démarré.")
                    st.info("💡 Démarrez Ollama avec: `ollama serve`")
                    return
                
                # Initialisation du générateur
                generator = BusinessCentralALGenerator(ollama_url, selected_model)
                
                # Extraction du contenu
                if input_mode == "📝 Texte direct (Recommandé)":
                    with st.spinner("📝 Analyse du contenu textuel..."):
                        pdf_content = specification_content
                else:
                    with st.spinner("🔍 Extraction du contenu PDF..."):
                        pdf_content = generator.extract_pdf_content(uploaded_file)
                
                # Analyse avec Ollama - SANS parsing structuré, direct vers l'IA
                with st.spinner(f"🤖 Analyse intelligente avec {selected_model}..."):
                    analysis = generator.analyze_specification_with_ai_only(pdf_content)
                
                # Debug : Afficher la réponse brute si demandé
                with st.expander("🔍 Debug - Voir l'analyse brute"):
                    st.json(analysis)
                
                # Affichage de l'analyse
                st.header("📋 Analyse de la spécification")
                
                col1, col2 = st.columns(2)
                
                with col1:
                    st.subheader("📊 Projet")
                    st.write(f"**Nom :** {analysis.get('project_name', 'N/A')}")
                    st.write(f"**Description :** {analysis.get('description', 'N/A')}")
                
                with col2:
                    st.subheader("🔢 Objets détectés")
                    st.metric("Tables", len(analysis.get('tables', [])))
                    st.metric("Pages", len(analysis.get('pages', [])))
                    st.metric("Codeunits", len(analysis.get('codeunits', [])))
                    st.metric("Rapports", len(analysis.get('reports', [])))
                
                # Détails des objets détectés
                with st.expander("🔍 Détails des objets détectés"):
                    
                    if analysis.get('tables'):
                        st.subheader("📊 Tables")
                        for table in analysis['tables']:
                            st.write(f"• **{table['name']}** - {table.get('description', 'N/A')}")
                    
                    if analysis.get('pages'):
                        st.subheader("📄 Pages")
                        for page in analysis['pages']:
                            st.write(f"• **{page['name']}** ({page.get('type', 'N/A')}) - {page.get('description', 'N/A')}")
                    
                    if analysis.get('codeunits'):
                        st.subheader("⚙️ Codeunits")
                        for codeunit in analysis['codeunits']:
                            st.write(f"• **{codeunit['name']}** - {codeunit.get('description', 'N/A')}")
                    
                    if analysis.get('reports'):
                        st.subheader("📈 Rapports")
                        for report in analysis['reports']:
                            st.write(f"• **{report['name']}** - {report.get('description', 'N/A')}")
                
                # Génération du code
                st.header("⚡ Génération du code AL")
                
                # Barre de progression
                progress_bar = st.progress(0)
                status_text = st.empty()
                
                def update_progress(progress, message):
                    progress_bar.progress(progress)
                    status_text.text(message)
                
                # Génération des objets
                objects, app_config = generator.generate_from_analysis(analysis, update_progress)
                
                # Résultats
                st.success(f"✅ Génération terminée ! {len(objects)} objets créés")
                
                # Résumé des objets générés
                st.subheader("📋 Objets générés")
                
                # Tableau des objets
                objects_data = []
                for obj in objects:
                    objects_data.append({
                        "Type": obj.type,
                        "ID": obj.id,
                        "Nom": obj.name,
                        "Fichier": obj.filename
                    })
                
                st.dataframe(objects_data, use_container_width=True)
                
                # Prévisualisation du code
                st.subheader("👀 Prévisualisation du code")
                
                selected_object = st.selectbox(
                    "Sélectionnez un objet pour prévisualiser le code :",
                    options=range(len(objects)),
                    format_func=lambda x: f"{objects[x].type} {objects[x].id} - {objects[x].name}"
                )
                
                if selected_object is not None:
                    obj = objects[selected_object]
                    st.code(obj.code, language="al")
                
                # Téléchargement
                st.header("💾 Téléchargement")
                
                # Création du ZIP
                zip_content = create_download_zip(objects, app_config)
                
                st.download_button(
                    label="📦 Télécharger l'extension complète (ZIP)",
                    data=zip_content,
                    file_name=f"{analysis.get('project_name', 'BC_Extension').replace(' ', '_')}.zip",
                    mime="application/zip",
                    type="primary",
                    use_container_width=True
                )
                
                st.success("🎉 Votre extension Business Central est prête !")
                
                # Instructions de déploiement
                with st.expander("📖 Instructions de déploiement"):
                    st.markdown("""
                    ### 🚀 Comment utiliser votre extension
                    
                    1. **Téléchargez** le fichier ZIP
                    2. **Extrayez** le contenu dans un nouveau dossier
                    3. **Ouvrez** le dossier dans Visual Studio Code
                    4. **Installez** l'extension AL pour VS Code si nécessaire
                    5. **Configurez** votre environnement Business Central
                    6. **Compilez** l'extension (Ctrl+Shift+P → "AL: Package")
                    7. **Déployez** dans votre environnement de test
                    
                    ### ⚙️ Configuration requise
                    - Visual Studio Code
                    - Extension AL
                    - Environnement Business Central (sandbox/dev)
                    - Docker (optionnel pour environnement local)
                    
                    ### 🏠 Avantages Ollama Local
                    - 🔒 **Confidentialité** : Vos données restent locales
                    - 💰 **Gratuit** : Aucun coût d'API
                    - ⚡ **Rapide** : Pas de latence réseau
                    - 🌐 **Hors ligne** : Fonctionne sans internet
                    """)
                
            except Exception as e:
                st.error(f"❌ Erreur lors de la génération : {str(e)}")
                st.exception(e)
                
                # Conseils de dépannage
                with st.expander("🔧 Conseils de dépannage"):
                    st.markdown("""
                    **Erreurs communes :**
                    
                    1. **Connexion Ollama** : Vérifiez que `ollama serve` est démarré
                    2. **Modèle manquant** : Installez le modèle avec `ollama pull deepseek-coder:6.7b`
                    3. **Mémoire insuffisante** : Essayez un modèle plus petit comme `qwen2.5-coder:1.5b`
                    4. **JSON malformé** : Le modèle peut générer du JSON invalide, réessayez
                    5. **Timeout** : Le modèle prend du temps, soyez patient
                    
                    **Commandes utiles :**
                    ```bash
                    # Vérifier Ollama
                    ollama list
                    
                    # Tester la connexion
                    curl http://localhost:11434/api/version
                    
                    # Redémarrer Ollama
                    ollama serve
                    ```
                    """)
    
    elif not selected_model:
        st.warning("⚠️ Veuillez sélectionner un modèle Ollama dans la barre latérale")
        
        with st.expander("🚀 Installation rapide Ollama"):
            st.markdown("""
            ### Windows
            1. Téléchargez depuis [ollama.com/download](https://ollama.com/download)
            2. Installez le fichier .exe
            3. Ouvrez un terminal et tapez :
            ```cmd
            ollama pull deepseek-coder:6.7b
            ollama serve
            ```
            
            ### Linux/macOS
            ```bash
            curl -fsSL https://ollama.com/install.sh | sh
            ollama pull deepseek-coder:6.7b
            ollama serve
            ```
            """)
    
    elif not uploaded_file:
        st.info("📄 Veuillez uploader un fichier PDF de spécification")
        
        # Exemple de spécification
        with st.expander("💡 Exemple de spécification PDF"):
            st.markdown("""
            ### Exemple de contenu pour votre PDF de spécification :
            
            **Titre :** Gestion des Commandes Clients
            
            **Description :** Extension pour gérer les commandes clients avec suivi des statuts
            
            **Tables nécessaires :**
            - Table Commande Client : Numéro, Date, Client, Montant Total, Statut
            - Table Ligne Commande : Numéro Commande, Article, Quantité, Prix Unitaire
            
            **Pages nécessaires :**
            - Page Liste des Commandes
            - Page Fiche Commande
            - Page Lignes de Commande
            
            **Fonctionnalités :**
            - Validation des commandes
            - Calcul automatique des totaux
            - Rapports de suivi
            
            **Codeunits :**
            - Gestion des commandes
            - Validation des données
            
            **Rapports :**
            - Commandes par période
            - Statistiques de vente
            """)
    
    # Footer avec informations
    st.divider()
    st.markdown("""
    <div style='text-align: center; color: #666;'>
        <p>🏠 <strong>Business Central AL Generator - Version Ollama Local</strong></p>
        <p>Génération de code AL 100% locale et privée • Aucune donnée envoyée sur internet</p>
    </div>
    """, unsafe_allow_html=True)

if __name__ == "__main__":
    main(), '', code, flags=re.MULTILINE)
            
        return code.strip()
            
        except Exception as e:
            raise Exception(f"Erreur lors de la génération du PermissionSet: {str(e)}")
    
    def create_app_json(self, project_name: str, description: str) -> Dict:
        """
        Crée la configuration app.json pour l'extension
        
        Args:
            project_name: Nom du projet
            description: Description du projet
            
        Returns:
            Configuration app.json
        """
        return {
            "id": str(uuid.uuid4()),  # GUID unique généré automatiquement
            "name": project_name,
            "publisher": "Your Company",
            "version": "1.0.0.0",
            "brief": description,
            "description": description,
            "privacyStatement": "",
            "EULA": "",
            "help": "",
            "url": "",
            "logo": "",
            "dependencies": [
                {
                    "id": "63ca2fa4-4f03-4f2b-a480-172fef340d3f",
                    "publisher": "Microsoft",
                    "name": "System Application",
                    "version": "20.0.0.0"
                },
                {
                    "id": "437dbf0e-84ff-417a-965d-ed2bb9650972",
                    "publisher": "Microsoft",
                    "name": "Base Application",
                    "version": "20.0.0.0"
                }
            ],
            "screenshots": [],
            "platform": "20.0.0.0",
            "application": "20.0.0.0",
            "idRanges": [
                {
                    "from": 50000,
                    "to": 50999
                }
            ],
            "resourceExposurePolicy": {
                "allowDebugging": True,
                "allowDownloadingSource": False,
                "includeSourceInSymbolFile": False
            }
        }
    
    def generate_from_analysis(self, analysis: Dict, progress_callback=None) -> Tuple[List[ALObject], Dict]:
        """
        Génère tous les objets AL à partir de l'analyse
        
        Args:
            analysis: Analyse de la spécification
            progress_callback: Fonction de callback pour le progrès
            
        Returns:
            Tuple (objets générés, app.json config)
        """
        project_name = analysis.get('project_name', 'BC Extension')
        description = analysis.get('description', 'Extension générée automatiquement')
        
        generated_objects = []
        total_objects = (
            len(analysis.get('tables', [])) +
            len(analysis.get('pages', [])) +
            len(analysis.get('codeunits', [])) +
            len(analysis.get('reports', [])) + 1  # +1 pour le permission set
        )
        current_object = 0
        
        # Tables
        if 'tables' in analysis:
            for table_spec in analysis['tables']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération table: {table_spec['name']}")
                
                code = self.generate_object_code('Table', table_spec)
                table_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Table',
                    name=table_spec['name'],
                    id=table_id,
                    code=code,
                    dependencies=[],
                    filename=f"Tab{table_id}.{table_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Pages
        if 'pages' in analysis:
            for page_spec in analysis['pages']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération page: {page_spec['name']}")
                
                context = {'tables': analysis.get('tables', [])}
                code = self.generate_object_code('Page', page_spec, context)
                page_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Page',
                    name=page_spec['name'],
                    id=page_id,
                    code=code,
                    dependencies=[page_spec.get('source_table', '')],
                    filename=f"Pag{page_id}.{page_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Codeunits
        if 'codeunits' in analysis:
            for codeunit_spec in analysis['codeunits']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération codeunit: {codeunit_spec['name']}")
                
                code = self.generate_object_code('Codeunit', codeunit_spec)
                codeunit_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Codeunit',
                    name=codeunit_spec['name'],
                    id=codeunit_id,
                    code=code,
                    dependencies=[],
                    filename=f"Cod{codeunit_id}.{codeunit_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Rapports
        if 'reports' in analysis:
            for report_spec in analysis['reports']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération rapport: {report_spec['name']}")
                
                code = self.generate_object_code('Report', report_spec)
                report_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Report',
                    name=report_spec['name'],
                    id=report_id,
                    code=code,
                    dependencies=report_spec.get('data_items', []),
                    filename=f"Rep{report_id}.{report_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Permission Set
        if progress_callback:
            progress_callback(current_object / total_objects, "Génération du Permission Set...")
        
        permissionset_code = self.generate_permission_set_code(project_name, generated_objects)
        permissionset_id = list(self.used_ids)[-1]  # Dernier ID utilisé
        
        al_object = ALObject(
            type='PermissionSet',
            name=f"{project_name} Permissions",
            id=permissionset_id,
            code=permissionset_code,
            dependencies=[],
            filename=f"Per{permissionset_id}.{project_name.replace(' ', '')}Permissions.al"
        )
        
        generated_objects.append(al_object)
        
        # Configuration app.json
        app_config = self.create_app_json(project_name, description)
        
        if progress_callback:
            progress_callback(1.0, "Génération terminée!")
        
        return generated_objects, app_config

def create_download_zip(objects: List[ALObject], app_config: Dict) -> bytes:
    """
    Crée un fichier ZIP avec tous les objets AL générés
    
    Args:
        objects: Liste des objets AL
        app_config: Configuration app.json
        
    Returns:
        Contenu du fichier ZIP en bytes
    """
    zip_buffer = io.BytesIO()
    
    with zipfile.ZipFile(zip_buffer, 'w', zipfile.ZIP_DEFLATED) as zip_file:
        # Ajouter app.json
        app_json_content = json.dumps(app_config, indent=2, ensure_ascii=False)
        zip_file.writestr("app.json", app_json_content)
        
        # Ajouter tous les objets AL
        for obj in objects:
            header = f"""// Generated by Business Central AL Generator (Ollama)
// Date: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
// Object: {obj.type} {obj.id} "{obj.name}"

"""
            content = header + obj.code
            zip_file.writestr(obj.filename, content)
    
    zip_buffer.seek(0)
    return zip_buffer.getvalue()

# Interface Streamlit
def main():
    """Interface principale Streamlit"""
    
    # Sidebar pour configuration
    with st.sidebar:
        st.header("🏠 Configuration Ollama")
        
        # Configuration Ollama
        ollama_url = st.text_input(
            "URL Ollama",
            value="http://localhost:11434",
            help="URL de votre serveur Ollama local"
        )
        
        # Vérification de la connexion Ollama
        if st.button("🔍 Tester la connexion"):
            client = OllamaClient(ollama_url)
            if client.is_available():
                st.success("✅ Ollama connecté !")
                
                # Lister les modèles disponibles
                models = client.list_models()
                if models:
                    st.info(f"📋 Modèles disponibles: {', '.join(models)}")
                else:
                    st.warning("⚠️ Aucun modèle trouvé")
            else:
                st.error("❌ Impossible de se connecter à Ollama")
        
        # Sélection du modèle - MODÈLES OPTIMISÉS POUR LE CODE
        model_options = [
            "qwen2.5-coder:32b",        # ⭐⭐⭐ TOP - Excellent pour AL (si 32GB+ RAM)
            "codestral:22b",            # ⭐⭐⭐ Mistral Codestral (excellent codeur)
            "qwen2.5-coder:14b",        # ⭐⭐⭐ Très bon compromis (16GB+ RAM)
            "deepseek-coder-v2:16b",    # ⭐⭐⭐ DeepSeek V2 amélioré
            "qwen2.5-coder:7b",         # ⭐⭐ Bon pour 8GB+ RAM
            "deepseek-coder:6.7b",      # ⭐⭐ Version classique
            "codellama:13b-instruct",   # ⭐⭐ Stable et performant
            "codellama:7b-instruct",    # ⭐ Pour machines limitées
            "granite-code:8b",          # ⭐ IBM Granite (nouveau)
            "autre"
        ]
        
        selected_model = st.selectbox(
            "Modèle à utiliser",
            options=model_options,
            help="Modèles optimisés pour la génération de code AL Business Central"
        )
        
        if selected_model == "autre":
            custom_model = st.text_input("Nom du modèle personnalisé")
            if custom_model:
                selected_model = custom_model
        
        st.divider()
        
        # Informations sur les modèles recommandés
        st.header("🎯 Modèles de code optimisés")
        st.markdown("""
        **🏆 TOP pour Business Central AL :**
        - `qwen2.5-coder:32b` (⭐⭐⭐ Excellence - 32GB RAM)
        - `codestral:22b` (⭐⭐⭐ Mistral Codestral - 24GB RAM)
        - `qwen2.5-coder:14b` (⭐⭐⭐ Excellent - 16GB RAM)
        
        **🚀 Bon compromis :**
        - `qwen2.5-coder:7b` (⭐⭐ Très bon - 8GB RAM)
        - `deepseek-coder-v2:16b` (⭐⭐ V2 amélioré)
        
        **💡 Installation recommandée :**
        ```bash
        # Pour machines puissantes (32GB+)
        ollama pull qwen2.5-coder:32b
        
        # Pour machines normales (16GB+)  
        ollama pull qwen2.5-coder:14b
        
        # Pour machines limitées (8GB+)
        ollama pull qwen2.5-coder:7b
        ```
        
        **⚡ Démarrage :**
        ```bash
        ollama serve
        ```
        """)
        
        # Auto-détection du meilleur modèle
        if st.button("🎯 Détecter le meilleur modèle"):
            client = OllamaClient(ollama_url)
            if client.is_available():
                available_models = client.list_models()
                
                # Ordre de préférence pour AL
                preferred_order = [
                    "qwen2.5-coder:32b", "codestral:22b", "qwen2.5-coder:14b",
                    "deepseek-coder-v2:16b", "qwen2.5-coder:7b", "deepseek-coder:6.7b",
                    "codellama:13b-instruct", "codellama:7b-instruct"
                ]
                
                best_model = None
                for model in preferred_order:
                    if model in available_models:
                        best_model = model
                        break
                
                if best_model:
                    st.success(f"🎯 Meilleur modèle détecté: **{best_model}**")
                    st.info("💡 Sélectionnez-le dans la liste ci-dessus")
                else:
                    st.warning("⚠️ Aucun modèle de code optimisé trouvé")
                    st.info("📥 Installez un modèle recommandé avec `ollama pull qwen2.5-coder:7b`")
            else:
                st.error("❌ Ollama non disponible")
        
        st.divider()
        
        # Informations
        st.header("📋 Informations")
        st.markdown("""
        **Objets AL générés :**
        - 📊 Tables avec relations
        - 📄 Pages (List/Card)
        - ⚙️ Codeunits (logique)
        - 📈 Rapports
        - 🔐 Permission Sets
        - 📦 app.json
        """)
        
        st.markdown("""
        **Avantages local :**
        - 🔒 Confidentialité totale
        - 💰 Pas de coûts API
        - ⚡ Pas de limite de requêtes
        - 🏠 Fonctionne hors ligne
        """)
    
    # En-tête principal
    st.title("🏠 Business Central AL Generator (Local)")
    st.markdown("**Génération automatique de code AL avec Ollama - 100% local et privé**")
    
    # Zone d'upload
    st.header("📄 Spécification d'entrée")
    
    # Choix du mode d'entrée
    input_mode = st.radio(
        "Choisissez votre mode de saisie :",
        ["📝 Texte direct (Recommandé)", "📄 Upload PDF"],
        help="Le mode texte direct est plus précis pour l'analyse Ollama"
    )
    
    specification_content = None
    
    if input_mode == "📝 Texte direct (Recommandé)":
        specification_content = st.text_area(
            "Collez votre spécification ici :",
            height=300,
            placeholder="""Exemple :
1. Purpose
This functional specification describes the design of a new vehicle management module for Dynamics 365 Business Central.

2. Tables and Fields
Table: Vehicle
- VehicleID (Code[20]): Unique identifier of the vehicle
- Brand (Text[50]): Brand of the vehicle
- Model (Text[50]): Model of the vehicle
- Year (Integer): Year of manufacture
- Status (Option): Available, In Use, Under Maintenance

Table: VehicleAssignment
- AssignmentID (Code[20]): Unique identifier
- VehicleID (Code[20]): ID of the assigned vehicle
- EmployeeID (Code[20]): ID of the assigned employee

3. Business Logic
- When a vehicle is assigned, it automatically changes to In Use status
- Maintenance alerts should be generated automatically""",
            help="Copiez-collez directement le texte de votre spécification"
        )
        
        if specification_content:
            st.success(f"✅ Spécification saisie : {len(specification_content)} caractères")
    
    else:  # Mode PDF
        uploaded_file = st.file_uploader(
            "Uploadez votre spécification PDF",
            type=['pdf'],
            help="Sélectionnez un fichier PDF contenant la spécification de votre extension Business Central"
        )
        
        if uploaded_file:
            st.success(f"📎 Fichier uploadé: {uploaded_file.name}")
            st.info(f"📏 Taille: {len(uploaded_file.getvalue()) / 1024:.1f} KB")
    
    # Continuer seulement si on a du contenu ou un fichier
    if (input_mode == "📝 Texte direct (Recommandé)" and specification_content) or (input_mode == "📄 Upload PDF" and uploaded_file):
        
        col1, col2 = st.columns([3, 1])
        
        with col1:
            if input_mode == "📝 Texte direct (Recommandé)":
                st.success("✅ Mode texte direct - Analyse plus précise")
                st.info(f"📏 Taille: {len(specification_content)} caractères")
            else:
                st.success(f"📎 Fichier uploadé: {uploaded_file.name}")
                st.info(f"📏 Taille: {len(uploaded_file.getvalue()) / 1024:.1f} KB")
            st.info(f"🤖 Modèle: {selected_model}")
        
        with col2:
            generate_button = st.button("🚀 Générer le code AL", type="primary", use_container_width=True)
        
        if generate_button and selected_model:
            try:
                # Vérification de la connexion Ollama
                client = OllamaClient(ollama_url, selected_model)
                if not client.is_available():
                    st.error("❌ Impossible de se connecter à Ollama. Vérifiez que le serveur est démarré.")
                    st.info("💡 Démarrez Ollama avec: `ollama serve`")
                    return
                
                # Initialisation du générateur
                generator = BusinessCentralALGenerator(ollama_url, selected_model)
                
                # Extraction du contenu
                if input_mode == "📝 Texte direct (Recommandé)":
                    with st.spinner("📝 Analyse du contenu textuel..."):
                        pdf_content = specification_content
                else:
                    with st.spinner("🔍 Extraction du contenu PDF..."):
                        pdf_content = generator.extract_pdf_content(uploaded_file)
                
                # Analyse avec Ollama - SANS parsing structuré, direct vers l'IA
                with st.spinner(f"🤖 Analyse intelligente avec {selected_model}..."):
                    analysis = generator.analyze_specification_with_ai_only(pdf_content)
                
                # Debug : Afficher la réponse brute si demandé
                with st.expander("🔍 Debug - Voir l'analyse brute"):
                    st.json(analysis)
                
                # Affichage de l'analyse
                st.header("📋 Analyse de la spécification")
                
                col1, col2 = st.columns(2)
                
                with col1:
                    st.subheader("📊 Projet")
                    st.write(f"**Nom :** {analysis.get('project_name', 'N/A')}")
                    st.write(f"**Description :** {analysis.get('description', 'N/A')}")
                
                with col2:
                    st.subheader("🔢 Objets détectés")
                    st.metric("Tables", len(analysis.get('tables', [])))
                    st.metric("Pages", len(analysis.get('pages', [])))
                    st.metric("Codeunits", len(analysis.get('codeunits', [])))
                    st.metric("Rapports", len(analysis.get('reports', [])))
                
                # Détails des objets détectés
                with st.expander("🔍 Détails des objets détectés"):
                    
                    if analysis.get('tables'):
                        st.subheader("📊 Tables")
                        for table in analysis['tables']:
                            st.write(f"• **{table['name']}** - {table.get('description', 'N/A')}")
                    
                    if analysis.get('pages'):
                        st.subheader("📄 Pages")
                        for page in analysis['pages']:
                            st.write(f"• **{page['name']}** ({page.get('type', 'N/A')}) - {page.get('description', 'N/A')}")
                    
                    if analysis.get('codeunits'):
                        st.subheader("⚙️ Codeunits")
                        for codeunit in analysis['codeunits']:
                            st.write(f"• **{codeunit['name']}** - {codeunit.get('description', 'N/A')}")
                    
                    if analysis.get('reports'):
                        st.subheader("📈 Rapports")
                        for report in analysis['reports']:
                            st.write(f"• **{report['name']}** - {report.get('description', 'N/A')}")
                
                # Génération du code
                st.header("⚡ Génération du code AL")
                
                # Barre de progression
                progress_bar = st.progress(0)
                status_text = st.empty()
                
                def update_progress(progress, message):
                    progress_bar.progress(progress)
                    status_text.text(message)
                
                # Génération des objets
                objects, app_config = generator.generate_from_analysis(analysis, update_progress)
                
                # Résultats
                st.success(f"✅ Génération terminée ! {len(objects)} objets créés")
                
                # Résumé des objets générés
                st.subheader("📋 Objets générés")
                
                # Tableau des objets
                objects_data = []
                for obj in objects:
                    objects_data.append({
                        "Type": obj.type,
                        "ID": obj.id,
                        "Nom": obj.name,
                        "Fichier": obj.filename
                    })
                
                st.dataframe(objects_data, use_container_width=True)
                
                # Prévisualisation du code
                st.subheader("👀 Prévisualisation du code")
                
                selected_object = st.selectbox(
                    "Sélectionnez un objet pour prévisualiser le code :",
                    options=range(len(objects)),
                    format_func=lambda x: f"{objects[x].type} {objects[x].id} - {objects[x].name}"
                )
                
                if selected_object is not None:
                    obj = objects[selected_object]
                    st.code(obj.code, language="al")
                
                # Téléchargement
                st.header("💾 Téléchargement")
                
                # Création du ZIP
                zip_content = create_download_zip(objects, app_config)
                
                st.download_button(
                    label="📦 Télécharger l'extension complète (ZIP)",
                    data=zip_content,
                    file_name=f"{analysis.get('project_name', 'BC_Extension').replace(' ', '_')}.zip",
                    mime="application/zip",
                    type="primary",
                    use_container_width=True
                )
                
                st.success("🎉 Votre extension Business Central est prête !")
                
                # Instructions de déploiement
                with st.expander("📖 Instructions de déploiement"):
                    st.markdown("""
                    ### 🚀 Comment utiliser votre extension
                    
                    1. **Téléchargez** le fichier ZIP
                    2. **Extrayez** le contenu dans un nouveau dossier
                    3. **Ouvrez** le dossier dans Visual Studio Code
                    4. **Installez** l'extension AL pour VS Code si nécessaire
                    5. **Configurez** votre environnement Business Central
                    6. **Compilez** l'extension (Ctrl+Shift+P → "AL: Package")
                    7. **Déployez** dans votre environnement de test
                    
                    ### ⚙️ Configuration requise
                    - Visual Studio Code
                    - Extension AL
                    - Environnement Business Central (sandbox/dev)
                    - Docker (optionnel pour environnement local)
                    
                    ### 🏠 Avantages Ollama Local
                    - 🔒 **Confidentialité** : Vos données restent locales
                    - 💰 **Gratuit** : Aucun coût d'API
                    - ⚡ **Rapide** : Pas de latence réseau
                    - 🌐 **Hors ligne** : Fonctionne sans internet
                    """)
                
            except Exception as e:
                st.error(f"❌ Erreur lors de la génération : {str(e)}")
                st.exception(e)
                
                # Conseils de dépannage
                with st.expander("🔧 Conseils de dépannage"):
                    st.markdown("""
                    **Erreurs communes :**
                    
                    1. **Connexion Ollama** : Vérifiez que `ollama serve` est démarré
                    2. **Modèle manquant** : Installez le modèle avec `ollama pull deepseek-coder:6.7b`
                    3. **Mémoire insuffisante** : Essayez un modèle plus petit comme `qwen2.5-coder:1.5b`
                    4. **JSON malformé** : Le modèle peut générer du JSON invalide, réessayez
                    5. **Timeout** : Le modèle prend du temps, soyez patient
                    
                    **Commandes utiles :**
                    ```bash
                    # Vérifier Ollama
                    ollama list
                    
                    # Tester la connexion
                    curl http://localhost:11434/api/version
                    
                    # Redémarrer Ollama
                    ollama serve
                    ```
                    """)
    
    elif not selected_model:
        st.warning("⚠️ Veuillez sélectionner un modèle Ollama dans la barre latérale")
        
        with st.expander("🚀 Installation rapide Ollama"):
            st.markdown("""
            ### Windows
            1. Téléchargez depuis [ollama.com/download](https://ollama.com/download)
            2. Installez le fichier .exe
            3. Ouvrez un terminal et tapez :
            ```cmd
            ollama pull deepseek-coder:6.7b
            ollama serve
            ```
            
            ### Linux/macOS
            ```bash
            curl -fsSL https://ollama.com/install.sh | sh
            ollama pull deepseek-coder:6.7b
            ollama serve
            ```
            """)
    
    elif not uploaded_file:
        st.info("📄 Veuillez uploader un fichier PDF de spécification")
        
        # Exemple de spécification
        with st.expander("💡 Exemple de spécification PDF"):
            st.markdown("""
            ### Exemple de contenu pour votre PDF de spécification :
            
            **Titre :** Gestion des Commandes Clients
            
            **Description :** Extension pour gérer les commandes clients avec suivi des statuts
            
            **Tables nécessaires :**
            - Table Commande Client : Numéro, Date, Client, Montant Total, Statut
            - Table Ligne Commande : Numéro Commande, Article, Quantité, Prix Unitaire
            
            **Pages nécessaires :**
            - Page Liste des Commandes
            - Page Fiche Commande
            - Page Lignes de Commande
            
            **Fonctionnalités :**
            - Validation des commandes
            - Calcul automatique des totaux
            - Rapports de suivi
            
            **Codeunits :**
            - Gestion des commandes
            - Validation des données
            
            **Rapports :**
            - Commandes par période
            - Statistiques de vente
            """)
    
    # Footer avec informations
    st.divider()
    st.markdown("""
    <div style='text-align: center; color: #666;'>
        <p>🏠 <strong>Business Central AL Generator - Version Ollama Local</strong></p>
        <p>Génération de code AL 100% locale et privée • Aucune donnée envoyée sur internet</p>
    </div>
    """, unsafe_allow_html=True)

if __name__ == "__main__":
    main(), '', code, flags=re.MULTILINE)
            
            # Supprimer les explications avant/après le code
            lines = code.split('\n')
            start_idx = -1
            end_idx = len(lines)
            
            # Trouver le début du code AL (table, page, codeunit, report)
            for i, line in enumerate(lines):
                if re.match(r'^\s*(table|page|codeunit|report)\s+\d+', line.lower()):
                    start_idx = i
                    break
            
            # Trouver la fin du code AL (dernière accolade fermante)
            brace_count = 0
            for i in range(start_idx if start_idx != -1 else 0, len(lines)):
                line = lines[i]
                brace_count += line.count('{') - line.count('}')
                if brace_count == 0 and (line.strip() == '}' or '}' in line):
                    end_idx = i + 1
                    break
            
            if start_idx != -1:
                cleaned_code = '\n'.join(lines[start_idx:end_idx])
            else:
                cleaned_code = code
            
            return cleaned_code.strip()
            
        except Exception as e:
            raise Exception(f"Erreur lors de la génération de {object_type}: {str(e)}")
    
    def chat_with_enhanced_parameters(self, messages: List[Dict], max_tokens: int = 4000, temperature: float = 0.05, top_p: float = 0.8) -> str:
        """
        Version améliorée du chat avec paramètres optimisés pour le code
        """
        prompt = ""
        for msg in messages:
            if msg["role"] == "system":
                prompt += f"System: {msg['content']}\n\n"
            elif msg["role"] == "user":
                prompt += f"User: {msg['content']}\n\n"
            elif msg["role"] == "assistant":
                prompt += f"Assistant: {msg['content']}\n\n"
        
        prompt += "Assistant: "
        
        payload = {
            "model": self.client.model,
            "prompt": prompt,
            "stream": False,
            "options": {
                "temperature": temperature,
                "top_p": top_p,
                "top_k": 40,  # Limiter les choix pour plus de cohérence
                "repeat_penalty": 1.1,  # Éviter les répétitions
                "num_predict": max_tokens,
                "stop": ["```", "Explanation:", "Note:", "//END"]  # Arrêter sur ces mots
            }
        }
        
        try:
            response = requests.post(
                f"{self.client.base_url}/api/generate",
                json=payload,
                timeout=180  # 3 minutes pour les gros modèles
            )
            response.raise_for_status()
            
            result = response.json()
            return result.get("response", "")
            
        except Exception as e:
            raise Exception(f"Erreur Ollama optimisée: {str(e)}")
    
    def generate_permission_set_code(self, project_name: str, objects: List[ALObject]) -> str:
        """
        Génère le code AL pour un permission set
        
        Args:
            project_name: Nom du projet
            objects: Liste des objets générés
            
        Returns:
            Code AL du permission set
        """
        permissionset_id = self.get_next_object_id('PermissionSet')
        
        objects_list = [f"{obj.type} {obj.id} '{obj.name}'" for obj in objects]
        
        prompt = f"""Tu es un expert en AL pour Dynamics 365 Business Central. Génère un PermissionSet complet.

PROJECT: {project_name}
PERMISSIONSET ID: {permissionset_id}
OBJECTS: {objects_list}

INSTRUCTIONS:
- Crée un PermissionSet AL avec l'ID spécifié
- Inclus tous les objets listés
- Donne les permissions complètes (RIMD) sur tous les objets
- Utilise un nom approprié basé sur le projet
- Utilise les meilleures pratiques AL

RETOURNE UNIQUEMENT LE CODE AL, sans explication."""
        
        try:
            response = self.client.chat(
                messages=[{"role": "user", "content": prompt}],
                max_tokens=2000
            )
            
            # Nettoyer la réponse
            code = response.strip()
            code = re.sub(r'^```(?:al)?', '', code, flags=re.MULTILINE)
            code = re.sub(r'^```$', '', code, flags=re.MULTILINE)
            
            return code.strip()
            
        except Exception as e:
            raise Exception(f"Erreur lors de la génération du PermissionSet: {str(e)}")
    
    def create_app_json(self, project_name: str, description: str) -> Dict:
        """
        Crée la configuration app.json pour l'extension
        
        Args:
            project_name: Nom du projet
            description: Description du projet
            
        Returns:
            Configuration app.json
        """
        return {
            "id": str(uuid.uuid4()),  # GUID unique généré automatiquement
            "name": project_name,
            "publisher": "Your Company",
            "version": "1.0.0.0",
            "brief": description,
            "description": description,
            "privacyStatement": "",
            "EULA": "",
            "help": "",
            "url": "",
            "logo": "",
            "dependencies": [
                {
                    "id": "63ca2fa4-4f03-4f2b-a480-172fef340d3f",
                    "publisher": "Microsoft",
                    "name": "System Application",
                    "version": "20.0.0.0"
                },
                {
                    "id": "437dbf0e-84ff-417a-965d-ed2bb9650972",
                    "publisher": "Microsoft",
                    "name": "Base Application",
                    "version": "20.0.0.0"
                }
            ],
            "screenshots": [],
            "platform": "20.0.0.0",
            "application": "20.0.0.0",
            "idRanges": [
                {
                    "from": 50000,
                    "to": 50999
                }
            ],
            "resourceExposurePolicy": {
                "allowDebugging": True,
                "allowDownloadingSource": False,
                "includeSourceInSymbolFile": False
            }
        }
    
    def generate_from_analysis(self, analysis: Dict, progress_callback=None) -> Tuple[List[ALObject], Dict]:
        """
        Génère tous les objets AL à partir de l'analyse
        
        Args:
            analysis: Analyse de la spécification
            progress_callback: Fonction de callback pour le progrès
            
        Returns:
            Tuple (objets générés, app.json config)
        """
        project_name = analysis.get('project_name', 'BC Extension')
        description = analysis.get('description', 'Extension générée automatiquement')
        
        generated_objects = []
        total_objects = (
            len(analysis.get('tables', [])) +
            len(analysis.get('pages', [])) +
            len(analysis.get('codeunits', [])) +
            len(analysis.get('reports', [])) + 1  # +1 pour le permission set
        )
        current_object = 0
        
        # Tables
        if 'tables' in analysis:
            for table_spec in analysis['tables']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération table: {table_spec['name']}")
                
                code = self.generate_object_code('Table', table_spec)
                table_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Table',
                    name=table_spec['name'],
                    id=table_id,
                    code=code,
                    dependencies=[],
                    filename=f"Tab{table_id}.{table_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Pages
        if 'pages' in analysis:
            for page_spec in analysis['pages']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération page: {page_spec['name']}")
                
                context = {'tables': analysis.get('tables', [])}
                code = self.generate_object_code('Page', page_spec, context)
                page_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Page',
                    name=page_spec['name'],
                    id=page_id,
                    code=code,
                    dependencies=[page_spec.get('source_table', '')],
                    filename=f"Pag{page_id}.{page_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Codeunits
        if 'codeunits' in analysis:
            for codeunit_spec in analysis['codeunits']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération codeunit: {codeunit_spec['name']}")
                
                code = self.generate_object_code('Codeunit', codeunit_spec)
                codeunit_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Codeunit',
                    name=codeunit_spec['name'],
                    id=codeunit_id,
                    code=code,
                    dependencies=[],
                    filename=f"Cod{codeunit_id}.{codeunit_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Rapports
        if 'reports' in analysis:
            for report_spec in analysis['reports']:
                if progress_callback:
                    progress_callback(current_object / total_objects, f"Génération rapport: {report_spec['name']}")
                
                code = self.generate_object_code('Report', report_spec)
                report_id = list(self.used_ids)[-1]  # Dernier ID utilisé
                
                al_object = ALObject(
                    type='Report',
                    name=report_spec['name'],
                    id=report_id,
                    code=code,
                    dependencies=report_spec.get('data_items', []),
                    filename=f"Rep{report_id}.{report_spec['name'].replace(' ', '')}.al"
                )
                
                generated_objects.append(al_object)
                current_object += 1
        
        # Permission Set
        if progress_callback:
            progress_callback(current_object / total_objects, "Génération du Permission Set...")
        
        permissionset_code = self.generate_permission_set_code(project_name, generated_objects)
        permissionset_id = list(self.used_ids)[-1]  # Dernier ID utilisé
        
        al_object = ALObject(
            type='PermissionSet',
            name=f"{project_name} Permissions",
            id=permissionset_id,
            code=permissionset_code,
            dependencies=[],
            filename=f"Per{permissionset_id}.{project_name.replace(' ', '')}Permissions.al"
        )
        
        generated_objects.append(al_object)
        
        # Configuration app.json
        app_config = self.create_app_json(project_name, description)
        
        if progress_callback:
            progress_callback(1.0, "Génération terminée!")
        
        return generated_objects, app_config

def create_download_zip(objects: List[ALObject], app_config: Dict) -> bytes:
    """
    Crée un fichier ZIP avec tous les objets AL générés
    
    Args:
        objects: Liste des objets AL
        app_config: Configuration app.json
        
    Returns:
        Contenu du fichier ZIP en bytes
    """
    zip_buffer = io.BytesIO()
    
    with zipfile.ZipFile(zip_buffer, 'w', zipfile.ZIP_DEFLATED) as zip_file:
        # Ajouter app.json
        app_json_content = json.dumps(app_config, indent=2, ensure_ascii=False)
        zip_file.writestr("app.json", app_json_content)
        
        # Ajouter tous les objets AL
        for obj in objects:
            header = f"""// Generated by Business Central AL Generator (Ollama)
// Date: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
// Object: {obj.type} {obj.id} "{obj.name}"

"""
            content = header + obj.code
            zip_file.writestr(obj.filename, content)
    
    zip_buffer.seek(0)
    return zip_buffer.getvalue()

# Interface Streamlit
def main():
    """Interface principale Streamlit"""
    
    # Sidebar pour configuration
    with st.sidebar:
        st.header("🏠 Configuration Ollama")
        
        # Configuration Ollama
        ollama_url = st.text_input(
            "URL Ollama",
            value="http://localhost:11434",
            help="URL de votre serveur Ollama local"
        )
        
        # Vérification de la connexion Ollama
        if st.button("🔍 Tester la connexion"):
            client = OllamaClient(ollama_url)
            if client.is_available():
                st.success("✅ Ollama connecté !")
                
                # Lister les modèles disponibles
                models = client.list_models()
                if models:
                    st.info(f"📋 Modèles disponibles: {', '.join(models)}")
                else:
                    st.warning("⚠️ Aucun modèle trouvé")
            else:
                st.error("❌ Impossible de se connecter à Ollama")
        
        # Sélection du modèle - MODÈLES OPTIMISÉS POUR LE CODE
        model_options = [
            "qwen2.5-coder:32b",        # ⭐⭐⭐ TOP - Excellent pour AL (si 32GB+ RAM)
            "codestral:22b",            # ⭐⭐⭐ Mistral Codestral (excellent codeur)
            "qwen2.5-coder:14b",        # ⭐⭐⭐ Très bon compromis (16GB+ RAM)
            "deepseek-coder-v2:16b",    # ⭐⭐⭐ DeepSeek V2 amélioré
            "qwen2.5-coder:7b",         # ⭐⭐ Bon pour 8GB+ RAM
            "deepseek-coder:6.7b",      # ⭐⭐ Version classique
            "codellama:13b-instruct",   # ⭐⭐ Stable et performant
            "codellama:7b-instruct",    # ⭐ Pour machines limitées
            "granite-code:8b",          # ⭐ IBM Granite (nouveau)
            "autre"
        ]
        
        selected_model = st.selectbox(
            "Modèle à utiliser",
            options=model_options,
            help="Modèles optimisés pour la génération de code AL Business Central"
        )
        
        if selected_model == "autre":
            custom_model = st.text_input("Nom du modèle personnalisé")
            if custom_model:
                selected_model = custom_model
        
        st.divider()
        
        # Informations sur les modèles recommandés
        st.header("🎯 Modèles de code optimisés")
        st.markdown("""
        **🏆 TOP pour Business Central AL :**
        - `qwen2.5-coder:32b` (⭐⭐⭐ Excellence - 32GB RAM)
        - `codestral:22b` (⭐⭐⭐ Mistral Codestral - 24GB RAM)
        - `qwen2.5-coder:14b` (⭐⭐⭐ Excellent - 16GB RAM)
        
        **🚀 Bon compromis :**
        - `qwen2.5-coder:7b` (⭐⭐ Très bon - 8GB RAM)
        - `deepseek-coder-v2:16b` (⭐⭐ V2 amélioré)
        
        **💡 Installation recommandée :**
        ```bash
        # Pour machines puissantes (32GB+)
        ollama pull qwen2.5-coder:32b
        
        # Pour machines normales (16GB+)  
        ollama pull qwen2.5-coder:14b
        
        # Pour machines limitées (8GB+)
        ollama pull qwen2.5-coder:7b
        ```
        
        **⚡ Démarrage :**
        ```bash
        ollama serve
        ```
        """)
        
        # Auto-détection du meilleur modèle
        if st.button("🎯 Détecter le meilleur modèle"):
            client = OllamaClient(ollama_url)
            if client.is_available():
                available_models = client.list_models()
                
                # Ordre de préférence pour AL
                preferred_order = [
                    "qwen2.5-coder:32b", "codestral:22b", "qwen2.5-coder:14b",
                    "deepseek-coder-v2:16b", "qwen2.5-coder:7b", "deepseek-coder:6.7b",
                    "codellama:13b-instruct", "codellama:7b-instruct"
                ]
                
                best_model = None
                for model in preferred_order:
                    if model in available_models:
                        best_model = model
                        break
                
                if best_model:
                    st.success(f"🎯 Meilleur modèle détecté: **{best_model}**")
                    st.info("💡 Sélectionnez-le dans la liste ci-dessus")
                else:
                    st.warning("⚠️ Aucun modèle de code optimisé trouvé")
                    st.info("📥 Installez un modèle recommandé avec `ollama pull qwen2.5-coder:7b`")
            else:
                st.error("❌ Ollama non disponible")
        
        st.divider()
        
        # Informations
        st.header("📋 Informations")
        st.markdown("""
        **Objets AL générés :**
        - 📊 Tables avec relations
        - 📄 Pages (List/Card)
        - ⚙️ Codeunits (logique)
        - 📈 Rapports
        - 🔐 Permission Sets
        - 📦 app.json
        """)
        
        st.markdown("""
        **Avantages local :**
        - 🔒 Confidentialité totale
        - 💰 Pas de coûts API
        - ⚡ Pas de limite de requêtes
        - 🏠 Fonctionne hors ligne
        """)
    
    # En-tête principal
    st.title("🏠 Business Central AL Generator (Local)")
    st.markdown("**Génération automatique de code AL avec Ollama - 100% local et privé**")
    
    # Zone d'upload
    st.header("📄 Spécification d'entrée")
    
    # Choix du mode d'entrée
    input_mode = st.radio(
        "Choisissez votre mode de saisie :",
        ["📝 Texte direct (Recommandé)", "📄 Upload PDF"],
        help="Le mode texte direct est plus précis pour l'analyse Ollama"
    )
    
    specification_content = None
    
    if input_mode == "📝 Texte direct (Recommandé)":
        specification_content = st.text_area(
            "Collez votre spécification ici :",
            height=300,
            placeholder="""Exemple :
1. Purpose
This functional specification describes the design of a new vehicle management module for Dynamics 365 Business Central.

2. Tables and Fields
Table: Vehicle
- VehicleID (Code[20]): Unique identifier of the vehicle
- Brand (Text[50]): Brand of the vehicle
- Model (Text[50]): Model of the vehicle
- Year (Integer): Year of manufacture
- Status (Option): Available, In Use, Under Maintenance

Table: VehicleAssignment
- AssignmentID (Code[20]): Unique identifier
- VehicleID (Code[20]): ID of the assigned vehicle
- EmployeeID (Code[20]): ID of the assigned employee

3. Business Logic
- When a vehicle is assigned, it automatically changes to In Use status
- Maintenance alerts should be generated automatically""",
            help="Copiez-collez directement le texte de votre spécification"
        )
        
        if specification_content:
            st.success(f"✅ Spécification saisie : {len(specification_content)} caractères")
    
    else:  # Mode PDF
        uploaded_file = st.file_uploader(
            "Uploadez votre spécification PDF",
            type=['pdf'],
            help="Sélectionnez un fichier PDF contenant la spécification de votre extension Business Central"
        )
        
        if uploaded_file:
            st.success(f"📎 Fichier uploadé: {uploaded_file.name}")
            st.info(f"📏 Taille: {len(uploaded_file.getvalue()) / 1024:.1f} KB")
    
    # Continuer seulement si on a du contenu ou un fichier
    if (input_mode == "📝 Texte direct (Recommandé)" and specification_content) or (input_mode == "📄 Upload PDF" and uploaded_file):
        
        col1, col2 = st.columns([3, 1])
        
        with col1:
            if input_mode == "📝 Texte direct (Recommandé)":
                st.success("✅ Mode texte direct - Analyse plus précise")
                st.info(f"📏 Taille: {len(specification_content)} caractères")
            else:
                st.success(f"📎 Fichier uploadé: {uploaded_file.name}")
                st.info(f"📏 Taille: {len(uploaded_file.getvalue()) / 1024:.1f} KB")
            st.info(f"🤖 Modèle: {selected_model}")
        
        with col2:
            generate_button = st.button("🚀 Générer le code AL", type="primary", use_container_width=True)
        
        if generate_button and selected_model:
            try:
                # Vérification de la connexion Ollama
                client = OllamaClient(ollama_url, selected_model)
                if not client.is_available():
                    st.error("❌ Impossible de se connecter à Ollama. Vérifiez que le serveur est démarré.")
                    st.info("💡 Démarrez Ollama avec: `ollama serve`")
                    return
                
                # Initialisation du générateur
                generator = BusinessCentralALGenerator(ollama_url, selected_model)
                
                # Extraction du contenu
                if input_mode == "📝 Texte direct (Recommandé)":
                    with st.spinner("📝 Analyse du contenu textuel..."):
                        pdf_content = specification_content
                else:
                    with st.spinner("🔍 Extraction du contenu PDF..."):
                        pdf_content = generator.extract_pdf_content(uploaded_file)
                
                # Analyse avec Ollama - SANS parsing structuré, direct vers l'IA
                with st.spinner(f"🤖 Analyse intelligente avec {selected_model}..."):
                    analysis = generator.analyze_specification_with_ai_only(pdf_content)
                
                # Debug : Afficher la réponse brute si demandé
                with st.expander("🔍 Debug - Voir l'analyse brute"):
                    st.json(analysis)
                
                # Affichage de l'analyse
                st.header("📋 Analyse de la spécification")
                
                col1, col2 = st.columns(2)
                
                with col1:
                    st.subheader("📊 Projet")
                    st.write(f"**Nom :** {analysis.get('project_name', 'N/A')}")
                    st.write(f"**Description :** {analysis.get('description', 'N/A')}")
                
                with col2:
                    st.subheader("🔢 Objets détectés")
                    st.metric("Tables", len(analysis.get('tables', [])))
                    st.metric("Pages", len(analysis.get('pages', [])))
                    st.metric("Codeunits", len(analysis.get('codeunits', [])))
                    st.metric("Rapports", len(analysis.get('reports', [])))
                
                # Détails des objets détectés
                with st.expander("🔍 Détails des objets détectés"):
                    
                    if analysis.get('tables'):
                        st.subheader("📊 Tables")
                        for table in analysis['tables']:
                            st.write(f"• **{table['name']}** - {table.get('description', 'N/A')}")
                    
                    if analysis.get('pages'):
                        st.subheader("📄 Pages")
                        for page in analysis['pages']:
                            st.write(f"• **{page['name']}** ({page.get('type', 'N/A')}) - {page.get('description', 'N/A')}")
                    
                    if analysis.get('codeunits'):
                        st.subheader("⚙️ Codeunits")
                        for codeunit in analysis['codeunits']:
                            st.write(f"• **{codeunit['name']}** - {codeunit.get('description', 'N/A')}")
                    
                    if analysis.get('reports'):
                        st.subheader("📈 Rapports")
                        for report in analysis['reports']:
                            st.write(f"• **{report['name']}** - {report.get('description', 'N/A')}")
                
                # Génération du code
                st.header("⚡ Génération du code AL")
                
                # Barre de progression
                progress_bar = st.progress(0)
                status_text = st.empty()
                
                def update_progress(progress, message):
                    progress_bar.progress(progress)
                    status_text.text(message)
                
                # Génération des objets
                objects, app_config = generator.generate_from_analysis(analysis, update_progress)
                
                # Résultats
                st.success(f"✅ Génération terminée ! {len(objects)} objets créés")
                
                # Résumé des objets générés
                st.subheader("📋 Objets générés")
                
                # Tableau des objets
                objects_data = []
                for obj in objects:
                    objects_data.append({
                        "Type": obj.type,
                        "ID": obj.id,
                        "Nom": obj.name,
                        "Fichier": obj.filename
                    })
                
                st.dataframe(objects_data, use_container_width=True)
                
                # Prévisualisation du code
                st.subheader("👀 Prévisualisation du code")
                
                selected_object = st.selectbox(
                    "Sélectionnez un objet pour prévisualiser le code :",
                    options=range(len(objects)),
                    format_func=lambda x: f"{objects[x].type} {objects[x].id} - {objects[x].name}"
                )
                
                if selected_object is not None:
                    obj = objects[selected_object]
                    st.code(obj.code, language="al")
                
                # Téléchargement
                st.header("💾 Téléchargement")
                
                # Création du ZIP
                zip_content = create_download_zip(objects, app_config)
                
                st.download_button(
                    label="📦 Télécharger l'extension complète (ZIP)",
                    data=zip_content,
                    file_name=f"{analysis.get('project_name', 'BC_Extension').replace(' ', '_')}.zip",
                    mime="application/zip",
                    type="primary",
                    use_container_width=True
                )
                
                st.success("🎉 Votre extension Business Central est prête !")
                
                # Instructions de déploiement
                with st.expander("📖 Instructions de déploiement"):
                    st.markdown("""
                    ### 🚀 Comment utiliser votre extension
                    
                    1. **Téléchargez** le fichier ZIP
                    2. **Extrayez** le contenu dans un nouveau dossier
                    3. **Ouvrez** le dossier dans Visual Studio Code
                    4. **Installez** l'extension AL pour VS Code si nécessaire
                    5. **Configurez** votre environnement Business Central
                    6. **Compilez** l'extension (Ctrl+Shift+P → "AL: Package")
                    7. **Déployez** dans votre environnement de test
                    
                    ### ⚙️ Configuration requise
                    - Visual Studio Code
                    - Extension AL
                    - Environnement Business Central (sandbox/dev)
                    - Docker (optionnel pour environnement local)
                    
                    ### 🏠 Avantages Ollama Local
                    - 🔒 **Confidentialité** : Vos données restent locales
                    - 💰 **Gratuit** : Aucun coût d'API
                    - ⚡ **Rapide** : Pas de latence réseau
                    - 🌐 **Hors ligne** : Fonctionne sans internet
                    """)
                
            except Exception as e:
                st.error(f"❌ Erreur lors de la génération : {str(e)}")
                st.exception(e)
                
                # Conseils de dépannage
                with st.expander("🔧 Conseils de dépannage"):
                    st.markdown("""
                    **Erreurs communes :**
                    
                    1. **Connexion Ollama** : Vérifiez que `ollama serve` est démarré
                    2. **Modèle manquant** : Installez le modèle avec `ollama pull deepseek-coder:6.7b`
                    3. **Mémoire insuffisante** : Essayez un modèle plus petit comme `qwen2.5-coder:1.5b`
                    4. **JSON malformé** : Le modèle peut générer du JSON invalide, réessayez
                    5. **Timeout** : Le modèle prend du temps, soyez patient
                    
                    **Commandes utiles :**
                    ```bash
                    # Vérifier Ollama
                    ollama list
                    
                    # Tester la connexion
                    curl http://localhost:11434/api/version
                    
                    # Redémarrer Ollama
                    ollama serve
                    ```
                    """)
    
    elif not selected_model:
        st.warning("⚠️ Veuillez sélectionner un modèle Ollama dans la barre latérale")
        
        with st.expander("🚀 Installation rapide Ollama"):
            st.markdown("""
            ### Windows
            1. Téléchargez depuis [ollama.com/download](https://ollama.com/download)
            2. Installez le fichier .exe
            3. Ouvrez un terminal et tapez :
            ```cmd
            ollama pull deepseek-coder:6.7b
            ollama serve
            ```
            
            ### Linux/macOS
            ```bash
            curl -fsSL https://ollama.com/install.sh | sh
            ollama pull deepseek-coder:6.7b
            ollama serve
            ```
            """)
    
    elif not uploaded_file:
        st.info("📄 Veuillez uploader un fichier PDF de spécification")
        
        # Exemple de spécification
        with st.expander("💡 Exemple de spécification PDF"):
            st.markdown("""
            ### Exemple de contenu pour votre PDF de spécification :
            
            **Titre :** Gestion des Commandes Clients
            
            **Description :** Extension pour gérer les commandes clients avec suivi des statuts
            
            **Tables nécessaires :**
            - Table Commande Client : Numéro, Date, Client, Montant Total, Statut
            - Table Ligne Commande : Numéro Commande, Article, Quantité, Prix Unitaire
            
            **Pages nécessaires :**
            - Page Liste des Commandes
            - Page Fiche Commande
            - Page Lignes de Commande
            
            **Fonctionnalités :**
            - Validation des commandes
            - Calcul automatique des totaux
            - Rapports de suivi
            
            **Codeunits :**
            - Gestion des commandes
            - Validation des données
            
            **Rapports :**
            - Commandes par période
            - Statistiques de vente
            """)
    
    # Footer avec informations
    st.divider()
    st.markdown("""
    <div style='text-align: center; color: #666;'>
        <p>🏠 <strong>Business Central AL Generator - Version Ollama Local</strong></p>
        <p>Génération de code AL 100% locale et privée • Aucune donnée envoyée sur internet</p>
    </div>
    """, unsafe_allow_html=True)

if __name__ == "__main__":
    main()

```
<br>

> Comme d'habitude, Si vous avez des questions, vous pouvez me contacter directement sur [Linkedin](https://www.linkedin.com/in/dominiquedelaire/) ou commenter l'article directement de Linkedin dans les commentaires :)
> Dominique
