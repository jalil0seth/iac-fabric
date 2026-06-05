# Fabric Platform IaC - Azure DevOps + Microsoft Fabric + ADLS Gen2

Ce package contient deux implémentations DevOps/IaC pour mettre en place une plateforme Azure/Fabric :

1. **Solution Bicep** : Azure resources en Bicep + Fabric workspace/Git/roles via scripts REST API.
2. **Solution Terraform** : Azure resources avec `azurerm`/`azuread` + Fabric workspace/roles/Git via provider `microsoft/fabric`.

## Architecture cible

- Resource Group dédié par environnement.
- ADLS Gen2 / Storage Account avec namespace hiérarchique.
- Conteneurs/layers : `landing`, `bronze`, `silver`, `gold`, `logs`, `archive`.
- Key Vault pour valeurs sensibles et références techniques.
- Microsoft Fabric Capacity F-SKU.
- Microsoft Fabric Workspace connecté à Azure DevOps Git.
- Groupes Entra ID pour gérer les accès : admins, contributors, viewers, data engineers.
- Azure DevOps pipeline avec Workload Identity Federation conseillé, sans secret client.

## Choix recommandé

Pour ton cas Azure Fabric + ADLS + Entra ID + Azure DevOps, **Terraform est recommandé** si tu veux gérer le workspace Fabric, les rôles Fabric et la connexion Git dans le même état IaC. Bicep reste très bon pour les ressources Azure natives, mais Fabric workspace et Git demandent encore souvent des appels REST/API ou scripts séparés.

## Contenu

```text
bicep/       Projet Bicep complet
terraform/   Projet Terraform complet
docs/        Rapport comparatif Bicep vs Terraform
```

## Pré-requis communs

- Un tenant Microsoft Entra ID.
- Une subscription Azure.
- Une organisation Azure DevOps et un repository.
- Un service connection Azure DevOps vers Azure, idéalement en Workload Identity Federation.
- Les permissions pour créer/modifier : resource groups, storage accounts, role assignments, Fabric capacity, Fabric workspace.
- Fabric Admin doit autoriser les service principals/managed identities si tu veux automatiser via identité applicative.

## Ordre de déploiement conseillé

1. Créer ou identifier les groupes Entra ID.
2. Créer le service connection Azure DevOps avec Workload Identity Federation.
3. Déployer la landing zone Azure : RG, ADLS Gen2, Key Vault, RBAC, Fabric Capacity.
4. Créer ou connecter le workspace Fabric.
5. Assigner les rôles Fabric aux groupes Entra ID.
6. Connecter le workspace Fabric à Azure DevOps Git.
7. Ajouter les pipelines de validation/deploy Fabric.
