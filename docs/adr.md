# Architecture Decision Record: Network Landing Zone

- **Status:** Proposed design baseline
- **Date:** 2026-09-27
- **Scope:** Standard network foundation for an MSP client environment
- **Implementation status:** Not yet published or deployed

## Context

A new Azure client workload should not begin from an empty subscription or a flat, undifferentiated VNet. The MSP needs a repeatable foundation that centralises shared services, keeps workload blast radius bounded, makes administrative access private, and carries its governance and secrets-management baseline with the deployment.

## Decision 1: Hub-spoke over a flat VNet or full mesh

The hub contains shared services such as Bastion, gateway connectivity, management services, and shared DNS integration. The spoke contains the client's workload subnets. Hub-spoke is preferred over a flat VNet because it separates shared control-plane services from workload blast radius. It is preferred over full mesh because adding a new spoke does not require a new peering relationship with every other spoke.

The topology is not a claim that every client requires identical address spaces. `clientName`, hub and spoke prefixes, and environment values remain parameters.

## Decision 2: Required subnet structure

The hub uses the Azure-required names `GatewaySubnet` and `AzureBastionSubnet`, plus `ManagementSubnet` and `SharedServicesSubnet`. The spoke uses `ApplicationSubnet` and `DatabaseSubnet`.

The required names are not cosmetic: Azure services reject alternate spellings for their dedicated subnets. The address ranges in the current baseline are defaults and must be checked for overlap with the client's existing networks before deployment.

## Decision 3: Private administration and traffic isolation

Workload VMs have no public IPs. Azure Bastion Developer SKU is the planned lab and demonstration management path. Application and database subnets receive different NSGs: application traffic is limited to the required web ports; database traffic is limited to the database port from the application subnet range. This makes the intended trust boundary visible in code and in the ADR.

The operational trade-off is explicit: removing public administration reduces exposure but makes Bastion and its supporting network path a dependency. The deployment and test plan must verify the path rather than treating the diagram as proof.

## Decision 4: Private endpoints and DNS

PaaS services that require private access use private endpoints in the workload network. The corresponding private DNS zones must be linked to every VNet that needs to resolve the private FQDN, including the hub and relevant spokes in the chosen topology. Public network access is disabled in the verification scenario so that a DNS or routing mistake cannot be hidden by a working public endpoint.

A service endpoint remains a valid choice for simpler, lower-compliance scenarios; it does not provide a private IP for the service and does not fully disable the public endpoint. The private endpoint decision is therefore tied to the client's isolation and compliance requirements.

## Decision 5: Key Vault and secrets

Key Vault uses RBAC authorization, public network access disabled, and private endpoint connectivity. The deployment does not accept secret values as Bicep parameters. A separate, authenticated operation writes secrets after the platform exists. Workload managed identities receive the narrow read role needed to retrieve secrets at boot or during an extension run.

This makes rotation a Key Vault operation rather than a parameter-file rewrite and prevents secret values from becoming part of deployment history or source control.

## Decision 6: Policy as part of the platform

The landing zone assigns governance rather than documenting governance as a suggestion. The planned SKU policy uses `Deny` for disallowed resource classes in production. The planned tag policy uses `Modify` for required environment and client tags where automatic compliance is preferable to blocking the deployment.

Policy controls what can be done; RBAC controls who can do it. Both are part of the platform because an MSP cannot reliably govern a client environment if the baseline depends on a later manual step.

## Planned resource set

- Hub VNet and four hub subnets.
- Spoke VNet and application/database subnets.
- Hub-spoke VNet peering with explicit peering properties.
- Bastion Developer SKU.
- NSGs for the workload subnets and any required management paths.
- Private endpoint and private DNS integration for the demonstration PaaS service.
- RBAC-mode Key Vault with private connectivity.
- User-assigned or system-assigned managed identity wiring as required by the final module contract.
- Resource-group-scoped SKU and tag policy assignments.

## Client deliverable

A completed implementation should provide a parameter file, deployable Bicep modules, architecture diagram, two-command deployment guidance, cleanup guidance, ADR, and observed verification evidence. A second client should be provisionable by changing client/address/environment parameters, not by copying and hand-editing the topology.

## Open decisions

- Final managed-identity attachment point for workload VMs and the exact secret bootstrap mechanism.
- Exact policy initiative identifiers and allowed SKU lists after validating current Azure policy definitions.
- Whether private DNS zones are centralised in the hub or linked through a separate shared-services subscription in a larger deployment.
- Current cost and SKU availability for every non-free resource before a test deployment.
