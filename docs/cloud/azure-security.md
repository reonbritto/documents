# Azure Security

## Azure Shared Responsibility Model

Like AWS, Azure follows a shared responsibility model where security duties are split between Microsoft and the customer, varying by service type (IaaS, PaaS, SaaS).

## Identity & Access Management

### Microsoft Entra ID (Azure AD)

```
Azure Identity Best Practices:
├── Enable MFA for all users
├── Use Conditional Access policies
├── Implement Privileged Identity Management (PIM)
├── Use managed identities for Azure resources
├── Disable legacy authentication protocols
└── Regular access reviews
```

### Conditional Access Policies

```json
{
  "displayName": "Require MFA for admin roles",
  "conditions": {
    "users": {
      "includeRoles": [
        "Global Administrator",
        "Security Administrator",
        "Exchange Administrator"
      ]
    },
    "applications": {
      "includeApplications": ["All"]
    }
  },
  "grantControls": {
    "operator": "AND",
    "builtInControls": [
      "mfa",
      "compliantDevice"
    ]
  }
}
```

### Managed Identities

```hcl
# Terraform: System-assigned managed identity
resource "azurerm_linux_virtual_machine" "app" {
  name = "app-vm"

  identity {
    type = "SystemAssigned"
  }
}

# Grant access to Key Vault
resource "azurerm_key_vault_access_policy" "app" {
  key_vault_id = azurerm_key_vault.main.id
  tenant_id    = data.azurerm_client_config.current.tenant_id
  object_id    = azurerm_linux_virtual_machine.app.identity[0].principal_id

  secret_permissions = ["Get", "List"]
}
```

## Network Security

### Network Security Groups (NSGs)

```hcl
resource "azurerm_network_security_group" "app" {
  name                = "app-nsg"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "AllowHTTPS"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "Internet"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "DenyAllInbound"
    priority                   = 4096
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}
```

## Data Protection

### Azure Key Vault

```bash
# Create Key Vault
az keyvault create \
  --name myapp-vault \
  --resource-group myapp-rg \
  --enable-purge-protection true \
  --enable-soft-delete true

# Store a secret
az keyvault secret set \
  --vault-name myapp-vault \
  --name "DatabasePassword" \
  --value "s3cure_p@ss"

# Retrieve in application (Python)
```

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()
client = SecretClient(
    vault_url="https://myapp-vault.vault.azure.net/",
    credential=credential
)

db_password = client.get_secret("DatabasePassword").value
```

## Azure Security Tools

| Tool | Purpose |
|------|---------|
| **Microsoft Defender for Cloud** | CSPM + CWP |
| **Microsoft Sentinel** | SIEM + SOAR |
| **Entra ID Protection** | Identity threat detection |
| **Azure Policy** | Compliance and governance |
| **Azure Key Vault** | Secret and key management |
| **Azure Firewall** | Network security |
| **Azure DDoS Protection** | DDoS mitigation |
| **Azure Monitor** | Logging and monitoring |

## Azure Policy (Compliance as Code)

```json
{
  "mode": "All",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.Storage/storageAccounts"
        },
        {
          "field": "Microsoft.Storage/storageAccounts/supportsHttpsTrafficOnly",
          "notEquals": "true"
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  }
}
```

## Azure Security Checklist

- [ ] MFA enforced for all users
- [ ] Conditional Access policies configured
- [ ] Privileged Identity Management enabled
- [ ] Microsoft Defender for Cloud enabled
- [ ] Network Security Groups on all subnets
- [ ] Key Vault for all secrets and keys
- [ ] Diagnostic logging enabled
- [ ] Azure Policy assignments for compliance
- [ ] Managed identities used (no stored credentials)
- [ ] Regular access reviews scheduled
