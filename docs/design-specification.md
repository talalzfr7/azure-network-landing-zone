# Design Specification: Network Landing Zone

## Parameters

| Parameter | Type | Purpose |
|---|---|---|
| `clientName` | `string` | Stable client/environment naming prefix. |
| `hubAddressSpace` | `string` | Hub VNet CIDR; planned default `10.0.0.0/16`. |
| `spokeAddressSpace` | `string` | Spoke VNet CIDR; planned default `10.1.0.0/16`. |
| `adminUsername` | `string` | Management VM account input. |
| `adminPassword` | `securestring` | Deployment-time secret; never committed. |
| `environmentType` | `string` | `dev`, `test`, or `prod`; controls policy values. |

## Module responsibilities

### `hub.bicep`

Creates the hub VNet and the four planned subnets:

- `GatewaySubnet` — `10.0.0.0/27`.
- `AzureBastionSubnet` — `10.0.1.0/26`.
- `ManagementSubnet` — `10.0.2.0/24`.
- `SharedServicesSubnet` — `10.0.3.0/24`.

The two service-reserved names must remain exact. The module must not silently reuse a client-overlapping address range.

### `spoke-workload.bicep`

Creates the spoke VNet and:

- `ApplicationSubnet` — `10.1.1.0/24`, with HTTP/HTTPS ingress rules as required by the workload.
- `DatabaseSubnet` — `10.1.2.0/24`, with database-port access only from the application subnet CIDR.
- Hub-spoke peering with the required forwarding and gateway properties.

### `bastion.bicep`

Deploys Bastion Developer SKU in the exact Bastion subnet. The module must document the Developer SKU's lab/demo constraints and the absence of a public-IP resource for that SKU.

### `keyvault.bicep`

Creates an RBAC-mode Key Vault with public network access disabled, private endpoint integration, and the narrow managed-identity role required for read-only secret retrieval. It accepts no secret values.

### `policy.bicep`

Assigns the validated built-in policy definitions or initiative selected for the project. The final implementation must make the `Deny` SKU effect and `Modify` tag effect explicit and testable.

### `main.bicep`

Orchestrates the modules in dependency order and exposes one deployable client-facing entry point.

## Verification matrix

| Behaviour | Evidence required |
|---|---|
| Peering | Both VNets show the intended peering state and a connectivity test succeeds. |
| Bastion | Management VM has no public IP and a Bastion connection succeeds. |
| NSG isolation | Application and database tests show allowed and denied paths as designed. |
| Private endpoint | Private DNS resolves to the endpoint IP from the linked VNet; public access is rejected when disabled. |
| Secrets | No secret in source/parameters; managed identity retrieves the test secret. |
| Policy | Disallowed SKU deployment is denied; missing tags reach the intended compliant state. |
| Cleanup | Disposable resource group and billable test resources are removed and checked. |

## Planned repository shape

```text
bicep/
├── bastion.bicep
├── hub.bicep
├── keyvault.bicep
├── main.bicep
├── policy.bicep
└── spoke-workload.bicep
architecture/
├── deployment-screenshot.png
└── landing-zone-architecture.png
docs/
├── adr.md
└── design-specification.md
```
