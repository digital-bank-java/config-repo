# Platform Configuration

Centralized, version-controlled, non-sensitive configuration for Digital Bank Java services.

## Purpose

This repository owns configuration consumed by Spring Cloud Config Server and its client services.

It contains:

- Shared platform defaults.
- Environment-specific platform overrides.
- Service-specific defaults.
- Environment-specific service overrides.

Credentials, cryptographic keys, tokens, customer information, and other sensitive values must not be stored in this repository.

## Repository Structure

```text
platform-config/
├── application.yml
├── application-local.yml
└── customer-service/
    ├── customer-service.yml
    └── customer-service-local.yml
```

- `application.yml` contains defaults shared by all services.
- `application-{profile}.yml` contains shared environment overrides.
- `{service}/{service}.yml` contains service defaults.
- `{service}/{service}-{profile}.yml` contains service environment overrides.

## Configuration Precedence

When the same property is defined in multiple files, the more specific source takes precedence. For a request such as `customer-service/local`, the effective order from highest to lowest priority is:

1. `customer-service/customer-service-local.yml`
2. `application-local.yml`
3. `customer-service/customer-service.yml`
4. `application.yml`

This allows shared defaults to remain stable while environments and individual services override only the values they need.

## Environment Model

| Profile | Purpose |
| --- | --- |
| `local` | A service running independently on a developer workstation. |
| `sit` | The integrated platform running in local Kubernetes. |
| `uat` | The AWS-hosted user acceptance environment. |
| `prod` | The AWS-hosted production environment. |

Environment-specific files are added as the corresponding environment becomes available. Production configuration must not reuse local or SIT endpoints or credentials.

## Secret Management

This repository must contain only non-sensitive configuration. Do not commit:

- Passwords or database credentials.
- API keys, access tokens, or OAuth client secrets.
- JWT signing keys, certificates with private keys, or TOTP secrets.
- Customer information, account identifiers, or payment data.
- Git credentials used by Config Server to access this repository.

For local and SIT workloads, synthetic secrets are supplied through local Kubernetes Secrets created from ignored local input. For UAT and production, secrets are stored in AWS Secrets Manager, encrypted with AWS KMS, and provided to workloads at runtime.

The `.gitignore` file reduces accidental additions of common secret-file formats, but it is not a security control. Every change must still be reviewed for sensitive content.

## Ownership and Contribution Workflow

Repository ownership is declared in `.github/CODEOWNERS`. All changes must be made on a dedicated branch and merged through a pull request; do not commit directly to `main`.

Recommended branch naming:

```text
feature/<issue-number>-<description>
fix/<issue-number>-<description>
docs/<issue-number>-<description>
```

Before opening a pull request:

1. Confirm that all YAML files are valid and correctly indented.
2. Review the effective precedence of every changed property.
3. Confirm that no secrets or customer data are present.
4. Verify the affected configuration through Config Server.
5. Reference the tracked issue using `Closes #<issue-number>` for an issue in this repository, or `Closes <owner>/<repository>#<issue-number>` for an issue in another repository.
