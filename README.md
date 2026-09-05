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
config-repo/
├── application.yml
├── application-sit.yml
├── account-service/
│   ├── account-service.yml
│   └── account-service-sit.yml
├── api-gateway/
│   ├── api-gateway.yml
│   └── api-gateway-sit.yml
├── auth-service/
│   ├── auth-service.yml
│   └── auth-service-sit.yml
├── customer-service/
│   ├── customer-service.yml
│   └── customer-service-sit.yml
├── ledger-service/
│   ├── ledger-service.yml
│   └── ledger-service-sit.yml
├── transaction-service/
│   ├── transaction-service.yml
│   └── transaction-service-sit.yml
├── mfa-service/
│   ├── mfa-service.yml
│   └── mfa-service-sit.yml
├── notification-service/
│   ├── notification-service.yml
│   └── notification-service-sit.yml
└── payment-service/
    ├── payment-service.yml
    └── payment-service-sit.yml
```

- `application.yml` contains defaults shared by all services.
- `application-{profile}.yml` contains shared environment overrides.
- `{service}/{service}.yml` contains service defaults.
- `{service}/{service}-{profile}.yml` contains service environment overrides.

## Configuration Precedence

When the same property is defined in multiple files, the more specific source takes precedence. For a request such as `customer-service/sit`, the effective order from highest to lowest priority is:

1. `customer-service/customer-service-sit.yml`
2. `application-sit.yml`
3. `customer-service/customer-service.yml`
4. `application.yml`

The same convention applies to `transaction-service/sit`; its service default
defines port `8084`, while its SIT file identifies the active runtime profile.
The same convention applies to `notification-service/sit`; its service default defines port `8088`, identity `notification-service`, and tier `security`, while its SIT file identifies the active runtime profile.

The shared `auth.jwt.issuer` identifies the local SIT token issuer for services
that validate Auth Service tokens. Auth Service adds the synthetic SIT scopes
`mfa.internal`, `payment.internal`, and `transfer.internal`; signing keys and
fixture credentials remain runtime secrets.

This allows shared defaults to remain stable while environments and individual services override only the values they need.

## SIT API Gateway Surface

The SIT API Gateway is the integration entry point for implemented service
workflows. Downstream services retain responsibility for authentication and
authorization; the gateway forwards the caller's `Authorization` header.

| Gateway path | Downstream service | Purpose |
| --- | --- | --- |
| `/api/v1/auth/**` | Auth Service | Login and logout |
| `/api/v1/mfa/**` | MFA Service | Enrollment and challenge workflows |
| `/internal/v1/transfer-workflows/**` | Transaction Service | Internal transfer workflow |
| `/internal/v1/payment-instructions/**` | Payment Service | Internal payment instruction lifecycle |

The internal transfer and payment paths are not public business APIs. They are
available through the authenticated SIT gateway for service integration and
verification only.

## API Gateway Resilience In SIT

The `api-gateway/api-gateway-sit.yml` file externalizes the non-secret SIT
resilience policy consumed by API Gateway:

- Retry up to two times for `GET`, `HEAD`, and `OPTIONS` requests only.
- Retry only on `5xx` and gateway-timeout responses, with bounded exponential
  backoff.
- Use a 1-second connection timeout and a 2-second downstream response timeout.
- Apply a per-route circuit breaker with a count-based window of 20 calls,
  opening at a 50% failure rate after at least 5 calls.
- Permit two calls during half-open recovery and transition automatically.

Mutation methods such as `POST`, `PUT`, `PATCH`, and `DELETE` are excluded from
automatic retries to avoid replaying non-idempotent business operations. The
policy is configuration-only; credentials, tokens, and production endpoints do
not belong in this repository.

The gateway resilience implementation and its focused integration coverage live
in the API Gateway repository. After changing these values, refresh or restart
Config Server and roll out API Gateway in SIT. Verify that an unavailable
downstream route is isolated and that unrelated routes remain available.

## Environment Model

| Profile | Purpose |
| --- | --- |
| `sit` | The integrated development and testing environment running in local Docker Desktop Kubernetes. |
| `uat` | The AWS-hosted user acceptance environment. |
| `prod` | The AWS-hosted production environment. |

`local` is not a supported deployment environment or Spring profile. A service can run from an IDE for debugging, but it connects to forwarded SIT dependencies and uses the `sit` profile with temporary workstation overrides.

Environment-specific files are added as the corresponding environment becomes available. Production configuration must not reuse SIT endpoints or credentials.

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

## Verification Through Config Server

This repository has no executable artifact. Verify configuration through the
deployed Config Server after its Git checkout has the intended commit:

```bash
kubectl port-forward service/config-server 18888:8888 --namespace digital-bank-sit
curl --fail http://localhost:18888/notification-service/default
curl --fail http://localhost:18888/notification-service/sit
curl --fail http://localhost:18888/payment-service/default
curl --fail http://localhost:18888/payment-service/sit
curl --fail http://localhost:18888/customer-service/sit
curl --fail http://localhost:18888/auth-service/sit
curl --fail http://localhost:18888/mfa-service/sit
```

Confirm that each response contains the intended property sources in
precedence order. For Payment Service, the SIT response must include the
service SIT override, `application-sit.yml`, the service default, and
`application.yml`; the effective configuration must include port `8085`,
service identity `payment-service`, tier `payments`, and runtime profile `sit`.
For Auth Service, verify issuer `digital-bank-auth` and the synthetic SIT
scopes. For MFA Service, verify issuer `digital-bank-auth` and runtime profile
`sit`.

After the configuration commit is available to the Config Server checkout,
verify the affected SIT workloads:

```bash
kubectl rollout status deployment/api-gateway --namespace digital-bank-sit --timeout=180s
kubectl rollout status deployment/notification-service --namespace digital-bank-sit --timeout=180s
kubectl get deployment,pods,service api-gateway notification-service --namespace digital-bank-sit
```

Verify the gateway routes and centralized documentation through the API Gateway
port-forward:

```bash
curl --fail http://localhost:8080/actuator/health
curl --fail http://localhost:8080/auth-service/actuator/health
curl --fail http://localhost:8080/mfa-service/actuator/health
curl --fail http://localhost:8080/transaction-service/actuator/health
curl --fail http://localhost:8080/payment-service/actuator/health
curl --fail http://localhost:8080/v3/api-docs/swagger-config
```

A missing downstream service should affect only its own route and must not make
unrelated gateway routes unavailable.

See the organization [README standard](https://github.com/digital-bank-java/.github/blob/main/docs/readme-standard.md) and [platform conventions](https://github.com/digital-bank-java/.github/blob/main/docs/platform-conventions.md) for shared documentation and naming rules.
