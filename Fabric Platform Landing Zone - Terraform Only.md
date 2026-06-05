# Fabric Platform Landing Zone - Terraform Only

Ce projet est une base **100% Terraform** pour créer une plateforme Microsoft Fabric + Azure Landing Zone ADLS Gen2 + Entra ID + Azure DevOps.

Il évite Bicep et garde une seule logique Infrastructure as Code.

## Ce que le projet déploie

1. **Azure Landing Zone Data**
   - Resource Group
   - ADLS Gen2 avec hierarchical namespace
   - Filesystems : `landing`, `bronze`, `silver`, `gold`, `checkpoints`, `config`
   - Dossiers standards pour les objets canoniques : `organisation`, `person`, `clients`, `chantiers`, `prestation`
   - Key Vault en RBAC mode
   - RBAC Azure sur Storage et Key Vault

2. **Microsoft Entra ID**
   - Groupes sécurité pour Fabric admins, engineers, readers, DevOps
   - Possibilité d'utiliser des groupes existants au lieu de les créer
   - Possibilité d'ajouter le service principal Azure DevOps/Terraform dans les groupes

3. **Microsoft Fabric**
   - Fabric Capacity F-SKU via AzureRM
   - Fabric Workspace via provider `microsoft/fabric`
   - Assignation des rôles Fabric Workspace aux groupes Entra ID
   - Git integration Azure DevOps optionnelle via `fabric_workspace_git`

4. **Azure DevOps**
   - Pipeline YAML Terraform prêt à l'emploi
   - Module Terraform optionnel pour créer Azure DevOps Project, repo et variable group

## Architecture cible

```text
Azure DevOps Pipeline / Service Principal
        |
        | Terraform providers: azurerm, azuread, fabric, azuredevops
        v
+-----------------------+       +-------------------------+
| Microsoft Entra ID    |       | Azure DevOps Project    |
| - Fabric Admins       |       | - Git repo              |
| - Data Engineers      |       | - Terraform pipeline    |
| - Readers             |       | - Variable group        |
+-----------+-----------+       +------------+------------+
            |                                |
            | RBAC / Workspace roles         | Git Integration
            v                                v
+-----------------------+       +-------------------------+
| Azure Data Landing    |       | Microsoft Fabric        |
| - ADLS2 landing       |       | - Capacity F SKU        |
| - Bronze/Silver/Gold  |       | - Workspace             |
| - Key Vault           |       | - Workspace roles       |
+-----------------------+       +-------------------------+
```

## Structure

```text
fabric-terraform-devops-complete/
├── .azuredevops/
│   └── azure-pipelines.yml
├── backend/
│   ├── main.tf
│   ├── variables.tf
│   └── terraform.tfvars.example
├── environments/
│   ├── dev/
│   ├── tst/
│   └── prod/
├── modules/
│   ├── entra/
│   ├── azure_landing_zone/
│   ├── fabric/
│   └── azure_devops_project/
├── scripts/
│   ├── bootstrap-backend.sh
│   ├── local-plan.sh
│   └── setup-fabric-tenant-settings.sh
└── docs/
    ├── access-model.md
    ├── runbook.md
    └── terraform-only-decision.md
```

## Pré-requis

- Terraform `>= 1.8`
- Azure CLI
- Accès Azure Subscription avec droits suffisants pour créer RG, Storage, Key Vault, Fabric Capacity et RBAC
- Droits Entra ID pour créer des groupes, ou fournir des groupes existants
- Un tenant Microsoft Fabric où les Service Principals/Managed Identities sont autorisés à utiliser les APIs Fabric
- Pour Azure DevOps provider : `AZDO_ORG_SERVICE_URL` et `AZDO_PERSONAL_ACCESS_TOKEN` si tu veux créer le projet/repo depuis Terraform

## Authentification recommandée

Pour un pipeline Azure DevOps, le mieux est :

- Service connection Azure Resource Manager avec **Workload Identity Federation**
- Service Principal ajouté dans un groupe Entra ID autorisé côté Fabric Admin Portal
- Pas de secret dans le repo

Pour test local rapide :

```bash
az login
az account set --subscription "<subscription-id>"

export ARM_SUBSCRIPTION_ID="<subscription-id>"
export ARM_TENANT_ID="<tenant-id>"
export ARM_CLIENT_ID="<app-client-id>"
export ARM_CLIENT_SECRET="<client-secret>"

export AZDO_ORG_SERVICE_URL="https://dev.azure.com/<org>"
export AZDO_PERSONAL_ACCESS_TOKEN="<pat>"
```

## Étape 1 - Créer le backend Terraform

```bash
cd backend
cp terraform.tfvars.example terraform.tfvars
terraform init
terraform apply
```

Ou avec le script :

```bash
./scripts/bootstrap-backend.sh \
  --subscription-id <subscription-id> \
  --resource-group rg-tfstate-fabric-frc \
  --storage-account sttfstatefabric001 \
  --container tfstate \
  --location francecentral
```

## Étape 2 - Configurer l'environnement DEV

```bash
cd environments/dev
cp terraform.tfvars.example terraform.tfvars
```

Puis modifier :

- `subscription_id`
- `tenant_id`
- `location`
- `project_code`
- `azure_devops_org_service_url`
- `azure_devops_project_name`
- `fabric_capacity_admin_members`
- `fabric_git_connection_id` si tu veux connecter Fabric à Azure DevOps par Terraform

## Étape 3 - Initialiser Terraform

```bash
terraform init \
  -backend-config="resource_group_name=rg-tfstate-fabric-frc" \
  -backend-config="storage_account_name=sttfstatefabric001" \
  -backend-config="container_name=tfstate" \
  -backend-config="key=fabric-platform-dev.tfstate"
```

## Étape 4 - Plan et Apply

```bash
terraform fmt -recursive
terraform validate
terraform plan -out tfplan
terraform apply tfplan
```

## Note importante sur Git Fabric

La ressource `fabric_workspace_git` est activée uniquement si :

```hcl
enable_fabric_git_integration = true
fabric_git_connection_id      = "<connection-id>"
```

Le `connection_id` correspond à une connexion Fabric/Azure DevOps déjà configurée ou créée dans Fabric. Cette partie est laissée optionnelle parce que l'intégration Git Fabric reste une zone plus sensible : elle dépend du mode d'authentification, du type de connexion Git et des droits Azure DevOps du principal de service.

## Recommandation

Pour ton cas, la version Terraform-only est la plus propre parce qu'elle permet de gérer dans le même code :

- Azure Resources
- Entra ID groups
- Azure RBAC
- Fabric workspace et rôles
- Azure DevOps project/repo/pipeline optionnel

Bicep est très bien pour Azure pur, mais il devient hybride pour Fabric workspace/roles/Git. Ici, le projet garde une gouvernance unique dans Terraform.
