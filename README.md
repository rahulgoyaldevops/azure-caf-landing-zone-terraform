<div align="center">

# 🏗️ Azure Monolithic Landing Zone — Terraform

[![Terraform](https://img.shields.io/badge/Terraform-v1.5+-623CE4?style=for-the-badge&logo=terraform&logoColor=white)](https://www.terraform.io/)
[![Azure](https://img.shields.io/badge/Azure-Cloud-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)](https://azure.microsoft.com/)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)
[![Terraform CI](https://github.com/rahulgoyaldevops/azure-caf-landing-zone-terraform/actions/workflows/terraform-ci.yml/badge.svg)](https://github.com/rahulgoyaldevops/azure-caf-landing-zone-terraform/actions/workflows/terraform-ci.yml)

**Enterprise-grade Azure Landing Zone built with modular Terraform — parent/child module pattern for scalable, reusable infrastructure conforming to Microsoft Cloud Adoption Framework (CAF).**

</div>

---

## 📐 Architecture
**Architecture Decision Record**: [ADR-001: Layered State Decomposition, Hub-Spoke Networking & CIS NSGs](docs/adr/001-layered-state-and-hub-spoke.md)


```
┌─────────────────────────────────────────────────────────────────────────┐
│                          PARENT MODULE                                  │
│                                                                         │
│  ┌───────────────────────┐                                              │
│  │  CAF Management Group │                                              │
│  │  Hierarchy Module     │ (Platform, Corp, Online, Sandboxes, Decomm)  │
│  └───────────────────────┘                                              │
│                                                                         │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                 │
│  │  Resource     │   │  Virtual     │   │   Subnet     │                 │
│  │  Group        │──▶│  Network     │──▶│              │                 │
│  │  Module       │   │  Module      │   │  Module      │                 │
│  └──────────────┘   └──────────────┘   └──────┬───────┘                 │
│                                                │                         │
│  ┌──────────────┐   ┌──────────────┐   ┌──────┴───────┐                 │
│  │  Route Table │   │  NSG Module  │   │  Linux VM    │                 │
│  │  (UDR 0.0.0.0│   │  (CIS 4096   │──▶│  + NIC Module│                 │
│  │   -> 10.100) │   │   Deny All)  │   │  (AZ-pinned) │                 │
│  └──────────────┘   └──────────────┘   └──────────────┘                 │
│                                                                         │
│  🔐 Secrets from Azure Key Vault (VM admin password)                    │
│  📦 Variable-driven: define infra in tfvars, not code                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 🌐 Hybrid Connectivity & Zero-Public-Exposure Private Link

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'curve': 'basis', 'nodeSpacing': 40, 'rankSpacing': 50}}}%%
flowchart TB
  subgraph T1["Tier 1: Hybrid Connectivity"]
    direction LR
    ONPREM["On-Premises Datacenter"] --> GW["ExpressRoute / S2S VPN Gateway"]
  end

  subgraph T2["Tier 2: Hub VNet"]
    direction LR
    FW["Azure Firewall / NVA"] --> UDR["UDR 0.0.0.0/0 Forced Tunneling"] --> DNS["Private DNS Resolver"]
  end

  subgraph T3["Tier 3: Peered Spoke VNets"]
    direction LR
    NSG["CIS 4096 Deny-All NSGs"] --> SUB["Workload Subnets"] --> PE["Private Endpoints"]
  end

  subgraph T4["Tier 4: Zero-Public-Exposure Azure PaaS"]
    direction LR
    PG["PostgreSQL Flexible Server"]
    KV["Key Vault"]
    ACR["ACR"]
    ST["Storage Account"]
  end

  T1 --> T2
  T2 --> T3
  T3 --> T4
  linkStyle default stroke:#64748B,stroke-width:1.5px

  classDef cobalt fill:#1D4ED8,stroke:#1E40AF,color:#FFFFFF,stroke-width:1.5px
  classDef slate fill:#0F172A,stroke:#334155,color:#FFFFFF,stroke-width:1.5px
  classDef emerald fill:#047857,stroke:#065F46,color:#FFFFFF,stroke-width:1.5px
  classDef teal fill:#0E7490,stroke:#155E75,color:#FFFFFF,stroke-width:1.5px
  classDef amber fill:#B45309,stroke:#92400E,color:#FFFFFF,stroke-width:1.5px

  class ONPREM slate
  class GW,FW,UDR,DNS cobalt
  class NSG amber
  class SUB teal
  class PE,PG,KV,ACR,ST emerald

  style T1 fill:transparent,stroke:#64748B,stroke-width:1.5px,color:#64748B
  style T2 fill:transparent,stroke:#64748B,stroke-width:1.5px,color:#64748B
  style T3 fill:transparent,stroke:#64748B,stroke-width:1.5px,color:#64748B
  style T4 fill:transparent,stroke:#64748B,stroke-width:1.5px,color:#64748B
```


## 🧩 Module Structure

```
tf-monolithic-landing-zone/
├── parentmodule/
│   ├── main.tf                    # Orchestrates all child modules
│   ├── variable.tf                # Input variables for all resources
│   ├── data.tf                    # Key Vault secret lookups
│   ├── provider.tf                # AzureRM provider config (v4.x)
│   ├── outputs.tf                 # Subnets, NSGs, Route Tables & MG IDs
│   ├── ARCHITECTURE_DEEP_DIVE.md  # Microsoft CAF Enterprise-Scale Guide
│   ├── .gitleaks.toml             # Secret scanning config
│   └── infracost.json             # Cost estimation output
│
└── child module/
    ├── management_group/           # CAF Management Groups (Platform, Corp, Online, Sandboxes)
    ├── resource_group/             # Azure Resource Group
    ├── virtual_network/            # VNet with configurable CIDR
    ├── subnet/                     # Subnet with delegation support
    ├── network_security_group/     # Granular NSG rules with default deny
    ├── route_table/                # UDR forced-tunneling (0.0.0.0/0 -> Azure Firewall)
    ├── pip/                        # Public IP allocation (Standard SKU)
    ├── network_interface/          # NIC with accelerated networking
    └── azurerm_linux_virtual_machine/ # Linux VM with Availability Zone pinning
```

## 🏛️ Microsoft Cloud Adoption Framework (CAF) Hierarchy

```
                                Tenant Root Group (/)
                                          │
                          Intermediate Root Management Group
                          (mg-enterprise / "Enterprise Root")
                                          │
    ┌───────────────────────┬─────────────┴─────────────┬────────────────────────┐
    ▼                       ▼                           ▼                        ▼
[Platform]            [Landing Zones]              [Sandboxes]            [Decommissioned]
(Management,          (Corp LZ,                    (Isolated R&D,         (Quarantine,
 Connectivity,        Online LZ)                   Zero Corp Peering)      Zero Resource Spends)
 Identity)
```

## 🚀 Key Design Decisions

| Decision | Rationale |
|:--|:--|
| **Microsoft CAF Compliance** | Management Group structure strictly separates Platform services from Corp/Online Workloads and Sandboxes |
| **Parent/Child Module Pattern** | Enforces separation of concerns — each resource type is independently versionable and testable |
| **User Defined Routing (UDR)** | Spoke subnets enforce `0.0.0.0/0` next-hop to Hub Azure Firewall (`10.100.1.4`) for zero-trust inspection |
| **Granular NSG Rules** | Subnets enforce explicit ingress allow lists and CIS-compliant priority 4096 Default Deny |
| **Availability Zone Resiliency**| VMs explicitly pinned to Availability Zones guaranteeing 99.99% infrastructure uptime SLA |
| **Variable-Driven Provisioning** | All infrastructure defined in `tfvars` — same modules deploy dev, UAT, and prod |
| **Key Vault Integration** | VM passwords fetched from Azure Key Vault at plan time — zero secrets in code or state |
| **Dependency Chaining** | Explicit `depends_on` ensures correct resource creation order |
| **Gitleaks Integration** | Pre-commit secret scanning prevents credential leaks |
| **Infracost Integration** | Cost estimation before apply — prevents cloud spend surprises |

## 💡 Usage

```bash
# Navigate to parent module
cd parentmodule

# Initialize Terraform
terraform init -backend=false

# Validate syntax & provider constraints
terraform validate

# Preview changes with cost estimation
terraform plan -var-file="environments/prod.tfvars"
```

## 🔐 Security Best Practices

- ✅ Microsoft CAF Enterprise-Scale hierarchy with policy boundaries
- ✅ Zero hardcoded secrets — Key Vault lookups via `data` blocks
- ✅ Pinned Availability Zones (`zone = "1"`)
- ✅ Network microsegmentation with subnet-level NSG bindings
- ✅ Forced tunneling egress via central Azure Firewall
- ✅ Gitleaks pre-commit scanning (`.gitleaks.toml`)
- ✅ State file excluded from git (`.gitignore`)
- ✅ Provider version pinning for reproducibility
---

<div align="center">

<sub>Designed & Implemented by <b><a href="https://github.com/rahulgoyaldevops">Rahul Goyal</a></b> — Senior DevOps & Cloud Platform Engineer (14+ Years Total IT)</sub>

<p align="center">
  <a href="https://www.linkedin.com/in/rahulgoyaldevops/" target="_blank">
    <img src="https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat-square&logo=linkedin&logoColor=white" alt="LinkedIn" />
  </a>
  &nbsp;
  <a href="https://github.com/rahulgoyaldevops?tab=repositories" target="_blank">
    <img src="https://img.shields.io/badge/GitHub-Repositories-181717?style=flat-square&logo=github&logoColor=white" alt="GitHub" />
  </a>
</p>

</div>
