# Rapport comparatif - Bicep vs Terraform pour Azure DevOps, Azure Fabric, ADLS Gen2 et Entra ID

## 1. Résumé exécutif

Pour une plateforme qui combine Azure DevOps, Microsoft Fabric, ADLS Gen2 et Entra ID, les deux solutions sont possibles mais elles ne couvrent pas le même périmètre avec le même niveau de maturité.

- **Bicep** est excellent pour les ressources Azure natives : resource group, storage account ADLS Gen2, Key Vault, RBAC Azure et Fabric Capacity.
- **Terraform** est plus adapté si l'objectif est de gérer aussi le workspace Fabric, les rôles Fabric et la connexion Git Azure DevOps dans une approche IaC déclarative et centralisée.

**Recommandation : Terraform** pour ce contexte, car il permet de piloter Azure + Entra + Fabric dans une même chaîne DevOps, avec un state partagé et une meilleure lisibilité pour les dépendances entre capacité, workspace, groupes et Git.

## 2. Périmètre fonctionnel demandé

| Besoin | Bicep | Terraform |
|---|---:|---:|
| Resource Group Azure | Oui | Oui |
| ADLS Gen2 / Storage Account | Oui | Oui |
| Containers landing/bronze/silver/gold | Oui | Oui |
| Key Vault | Oui | Oui |
| Azure RBAC sur ADLS | Oui | Oui |
| Groupes Entra ID | Possible via Microsoft Graph Bicep extension, mais plus sensible | Oui via provider azuread |
| Fabric Capacity F-SKU | Oui via Microsoft.Fabric/capacities | Oui via azurerm_fabric_capacity |
| Fabric Workspace | REST API/script recommandé | Oui via microsoft/fabric provider |
| Workspace roles Admin/Contributor/Viewer | REST API/script | Oui via fabric_workspace_role_assignment |
| Connexion Git Azure DevOps vers Fabric | REST API/script | Oui via fabric_workspace_git, avec attention aux credentials |
| Un seul state IaC | Non complet | Oui |

## 3. Analyse Bicep

### Points forts

- Très naturel pour Azure : même syntaxe que ARM, intégration Azure CLI native.
- Bonne lisibilité pour les équipes Microsoft/Azure.
- Très adapté si ton standard entreprise est Azure-only.
- Les modules sont simples pour ADLS, Key Vault, RBAC Azure et Fabric Capacity.

### Limites

- Le workspace Fabric, ses rôles et la connexion Git ne sont pas encore le point fort du Bicep pur.
- Il faut ajouter des scripts PowerShell/REST API après le déploiement Bicep.
- La gestion des groupes Entra via Microsoft Graph Bicep extension est possible mais demande des permissions Graph et comporte encore des limites.
- Le résultat est hybride : Bicep + PowerShell + REST API.

### Complexité

- Code Azure : simple à moyen.
- Fabric automation : moyen à élevé car elle sort du modèle Bicep.
- Maintenance : bonne pour Azure, plus fragile pour Fabric.

## 4. Analyse Terraform

### Points forts

- Plus cohérent pour gérer Azure + Entra + Fabric dans le même pipeline.
- Le provider `azuread` gère les groupes Entra ID.
- Le provider `microsoft/fabric` gère workspace, roles et Git integration.
- Un seul Terraform state donne une vue claire des dépendances.
- Plus adapté aux environnements multiples dev/test/prod.

### Limites

- Le provider Fabric est plus jeune que le provider AzureRM.
- Certaines opérations Git Fabric peuvent dépendre du type d'identité, de la configuration Fabric tenant et des credentials Git.
- Il faut une bonne gouvernance du Terraform state.
- Nécessite de bien gérer les versions provider.

### Complexité

- Code Azure : moyen.
- Fabric automation : plus simple que Bicep, car déclaratif.
- Maintenance : meilleure si l'équipe accepte Terraform.

## 5. Différence de philosophie

### Bicep

Bicep est le langage natif Azure. Il est idéal pour décrire des ressources Azure dans un abonnement. Quand le besoin reste centré sur Azure Resource Manager, Bicep est simple, direct et robuste.

### Terraform

Terraform est multi-provider. Pour ton besoin, ce point est important car la plateforme touche plusieurs plans : Azure Resource Manager, Microsoft Entra ID et Microsoft Fabric. Terraform peut coordonner ces plans dans un seul graphe de dépendances.

## 6. Quelle solution est la mieux adaptée ?

### Si l'objectif est une landing zone Azure uniquement

Choisir **Bicep**.

Exemple : RG, ADLS2, Key Vault, RBAC Azure, Fabric Capacity et configuration Fabric manuelle ou via scripts séparés.

### Si l'objectif est une plateforme Fabric automatisée de bout en bout

Choisir **Terraform**.

Exemple : groupes Entra, RBAC Azure, Fabric Capacity, Fabric Workspace, rôles workspace et connexion Git Azure DevOps.

## 7. Recommandation pour ton projet

Je recommande de partir sur **Terraform comme solution principale**, avec Bicep comme alternative Azure-native.

Raison : ton besoin n'est pas seulement Azure. Tu veux aussi connecter Azure DevOps au workspace Fabric, gérer les accès par groupes Entra ID et industrialiser le déploiement. Terraform est plus aligné avec cette orchestration complète.

## 8. Architecture de livraison recommandée

- Repo Azure DevOps unique : `fabric-platform-iac`.
- Branches : `main`, `develop`, feature branches.
- Environnements : `dev`, `test`, `prod`.
- Service connection Azure DevOps avec Workload Identity Federation.
- Groupes Entra ID comme point d'abstraction des accès.
- Terraform remote state sur Storage Account sécurisé.
- Pipeline avec étapes : fmt, validate, plan, manual approval, apply.
- Fabric workspace connecté au repo Git.

## 9. Décision finale

| Critère | Meilleur choix |
|---|---|
| Simplicité Azure native | Bicep |
| Couverture Azure + Entra + Fabric | Terraform |
| IaC de bout en bout | Terraform |
| Gouvernance state et drift | Terraform |
| Équipe 100% Microsoft Azure | Bicep |
| Industrialisation multi-environnement | Terraform |

**Conclusion : Terraform est le meilleur choix pour ton cas.** Bicep reste utile si le client impose Azure-native ou si la plateforme Fabric est configurée séparément.

## 10. Sources utiles

- Microsoft Learn - Microsoft.Fabric/capacities ARM/Bicep.
- Microsoft Learn - Fabric REST API Workspaces.
- Microsoft Learn - Fabric REST API Workspace role assignments.
- Microsoft Learn - Fabric REST API Git connect.
- Microsoft Learn - Fabric Git integration with Azure DevOps.
- Microsoft Learn - Azure DevOps service connections and Workload Identity Federation.
- Terraform Registry - microsoft/fabric provider.
- Terraform Registry - azurerm_fabric_capacity.
