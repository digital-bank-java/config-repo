# AGENTS.md

## Repository Purpose

`config-repo` stores runtime configuration served by `config-server`.

This is configuration data, not application code.

## What Belongs Here

- shared Spring config defaults
- shared profile overrides
- service-specific defaults
- service-specific profile overrides

## What Does Not Belong Here

- Java source code
- Helm manifests
- Dockerfiles
- secrets in plain text
- infrastructure bootstrapping

## File Model

- `application.yml`: shared defaults
- `application-{profile}.yml`: shared profile overrides
- `{service}/{service}.yml`: service defaults
- `{service}/{service}-{profile}.yml`: service profile overrides

## Current Services

- `customer-service`
- `account-service`
- `ledger-service`
- `api-gateway`
- `transaction-service`
- `auth-service`
- `mfa-service`

## Working Rules

- Keep configuration names aligned with Spring Config conventions.
- Use placeholders for secrets and credentials.
- Do not rename the repository structure casually; downstream assumptions depend on it.
- Keep service ownership clear: each service only gets its own runtime values here.

## Verification

Typical verification is indirect:

- run or query `config-server`
- request `/{service}/{profile}`
- confirm precedence and property source order

## Dependency Notes

- `config-server` depends on this repository.
- Most platform services depend on values served from this repository in SIT and above.
