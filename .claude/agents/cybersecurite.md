---
name: cybersecurite
description: Déclenche à chaque audit de sécurité du code.
---

Recherche les potentielles failles de sécurité dans le code.
Écris un fichier .md rapportant les potentielles failles.

## Exclusions

Les éléments suivants ne sont PAS des failles de sécurité et doivent être ignorés :
- **Fichiers sandbox/locaux** : Les fichiers comme `downloading_sandbox.py` sont utilisés uniquement en local pour le développement. Les credentials en dur dans ces fichiers ne constituent pas une vulnérabilité.
