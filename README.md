# Azure Network Landing Zone

> A hub-spoke Bicep design for repeatable MSP client environment provisioning.

**Status: design/specification baseline.** This repository records the landing-zone architecture, security and governance decisions, module contracts, and verification criteria. The Bicep implementation and deployment evidence are intentionally not published yet.

## Client scenario

An MSP needs a standard network foundation for each new Azure client environment. Shared services belong in a hub; client workloads belong in an isolated spoke. The baseline should provide private administration, predictable subnet isolation, private PaaS access, DNS integration, secrets management, and governance enforcement without relying on a junior engineer to remember a checklist after deployment.

## Planned topology

```text
Hub VNet: 10.0.0.0/16
├── GatewaySubnet          10.0.0.0/27
├── AzureBastionSubnet     10.0.1.0/26
├── ManagementSubnet       10.0.2.0/24
└── SharedServicesSubnet   10.0.3.0/24
              │
              │ VNet peering with explicit forwarding/gateway properties
              │
Spoke VNet: 10.1.0.0/16
├── ApplicationSubnet     10.1.1.0/24
└── DatabaseSubnet        10.1.2.0/24
```

The topology is a reusable baseline, not a claim that every client needs these exact prefixes. Address spaces and client naming are parameters; the required Azure subnet names remain exact where Azure requires them.

## Planned modules

| Module | Responsibility |
|---|---|
| `bicep/hub.bicep` | Hub VNet, required shared-service subnets, and hub-side security baseline. |
| `bicep/spoke-workload.bicep` | Spoke VNet, application/database subnets, differentiated NSGs, and peering. |
| `bicep/bastion.bicep` | Azure Bastion Developer SKU in `AzureBastionSubnet`. |
| `bicep/keyvault.bicep` | RBAC-mode Key Vault with public access disabled and private endpoint integration. |
| `bicep/policy.bicep` | SKU and tag governance at the resource-group scope. |
| `bicep/main.bicep` | Orchestrates all modules for one client environment. |

## Security and governance design

- Workload VMs have no public IPs; administration uses Bastion.
- Application and database subnets have different NSG rules. Database access is restricted to the application subnet on the required database port.
- PaaS access uses private endpoints and the corresponding private DNS zone links; disabling public network access is part of the intended verification.
- Key Vault uses RBAC authorization, public network access disabled, and private connectivity. Secret values are written separately by an operator or orchestration script; they are not Bicep parameters.
- The planned SKU policy uses `Deny` where cost/resource-class enforcement must block a deployment. The planned tag policy uses `Modify` where compliance should be repaired rather than reject an otherwise valid deployment.
- Policy and RBAC remain distinct: Policy constrains what can be deployed; RBAC controls who can perform actions.

## Verification criteria

Implementation will not be considered complete until the following are demonstrated and documented:

1. Each Bicep module and the orchestration template compile.
2. Hub and spoke peering works with the intended forwarding and gateway settings.
3. Bastion connects to a management VM with no public IP.
4. Application and database NSG isolation behaves as designed.
5. A private endpoint resolves through the linked private DNS zone and public access is rejected when disabled.
6. Key Vault rejects public access and a managed identity retrieves a secret without a secret in the deployment manifest.
7. The SKU policy blocks a disallowed deployment and the tag policy produces the intended compliance state.
8. The disposable test resource group and credit-consuming resources are deleted and cleanup is verified.

## Evidence boundary

This repository currently proves that the landing-zone design has been specified. It does **not** yet prove that the Bicep compiles, that the topology has been deployed, or that the four verification points have passed.

## AZ-104 and MSP relevance

The project maps the AZ-104 networking domain to the standard MSP onboarding problem: provide a repeatable, segmented, private-by-default environment with a governance baseline that survives handoff.

- Sprint scope: SD-41, SD-44, SD-45, SD-47, SD-48, and SD-49.
- Exam domain: Implement and manage virtual networking.
- Design references: [Hub-spoke reference architecture](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/hub-spoke), [private endpoint DNS integration](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns-integration), [Azure Bastion](https://learn.microsoft.com/en-us/azure/bastion/bastion-overview), [Key Vault overview](https://learn.microsoft.com/en-us/azure/key-vault/general/overview), and [Policy effects](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effects).
- Study-plan source: [AZ-104 sprint repository](https://github.com/talalzfr7/az104-sprint).

See [`docs/adr.md`](docs/adr.md) for the design decisions and [`docs/design-specification.md`](docs/design-specification.md) for the module and verification contract.
