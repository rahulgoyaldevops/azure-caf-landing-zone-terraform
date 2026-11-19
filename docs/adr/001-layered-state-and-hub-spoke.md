# ADR-001: Layered Terraform State Decomposition, Hub-Spoke Networking, and CIS Zero-Trust NSGs

## Status
Accepted

## Context
Enterprise Azure Cloud Adoption Framework (CAF) landing zones require clear blast-radius isolation, predictable network egress/inspection routing, and baseline zero-trust network security controls aligned with CIS Azure Benchmarks.

## Decisions

### 1. Layered Terraform State Decomposition
Rather than managing the entire landing zone in a single monolithic state file, infrastructure is decomposed into four independent state layers:
- **01-management**: Management groups, Azure Policy assignments, diagnostic settings, and centralized Log Analytics workspaces.
- **02-connectivity**: Hub VNet, Azure Firewall / NVA, Bastion, VPN/ExpressRoute gateways, Private DNS zones, and core route tables.
- **03-identity**: Identity workloads, Active Directory Domain Services / Entra ID integration components, and dedicated identity subnets.
- **04-workloads**: Application landing zones, spoke VNets, workload-specific PaaS/compute resources, and VNet peerings to the hub.

**Rationale**: Minimizes blast radius, reduces `terraform plan`/`apply` execution times, enables independent RBAC and CI/CD pipelines per platform team, and prevents workload changes from accidentally impacting core connectivity or governance.

### 2. Hub-Spoke Topology with UDR Forced Tunneling vs. Azure Virtual WAN
We selected a traditional **Hub-Spoke topology with User-Defined Routes (UDRs) for forced tunneling** (`0.0.0.0/0` next-hop Azure Firewall / NVA) over **Azure Virtual WAN (vWAN)**.

**Trade-offs**:
- **Hub-Spoke + UDRs (Chosen)**: Full deterministic control over routing tables, explicit inspection of east-west and north-south traffic, lower baseline cost for single-region or moderate multi-region footprints, and direct Terraform resource transparency. Requires manual management of UDRs and peering topologies as spokes scale.
- **Azure Virtual WAN**: Simplifies large-scale global transit, automated branch-to-branch routing, and hub-to-hub mesh connectivity, at the expense of higher fixed costs, reduced route-table granularity in some scenarios, and less direct control over custom UDR overrides.

### 3. CIS 4096 Deny-All Network Security Groups (NSGs)
All workload and platform subnets enforce an explicit **priority 4096 deny-all inbound and outbound rule** (`DenyAllInbound` / `DenyAllOutbound`) at the bottom of each NSG.

**Rationale**: Overrides Azure's permissive default VNet-to-VNet and Internet outbound rules (`AllowVnetInBound`, `AllowInternetOutBound`), establishing a zero-trust allow-list posture aligned with CIS Microsoft Azure Foundations Benchmark recommendations. Only explicitly permitted ports, protocols, and CIDR/service-tag flows at higher priorities are allowed.

## Consequences
- Cross-layer dependencies are consumed via remote state data sources (`terraform_remote_state`) or explicit Azure data lookups.
- Every new spoke subnet must define explicit allow rules above priority 4096 and associate the forced-tunneling route table.
