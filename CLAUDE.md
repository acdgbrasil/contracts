# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repositório

Contratos de API da **ACDG Technology** — organização sem fins lucrativos que desenvolve tecnologia para pacientes com doenças genéticas raras. Este repositório define os contratos de integração (OpenAPI para sync, AsyncAPI para async) e modelos canônicos de domínio consumidos pelos serviços do monorepo.

Atualmente apenas o serviço `social-care` possui contratos definidos.

## Validação e CI

```bash
# Lint de specs OpenAPI (requer Node.js)
npx @redocly/cli@latest lint services/social-care/openapi/openapi.yaml

# Validação de specs AsyncAPI
npx @asyncapi/cli@latest validate services/social-care/asyncapi/asyncapi.yaml
```

O CI (`.github/workflows/contracts-bundle.yml`) executa lint OpenAPI e validação AsyncAPI em PRs para `main`. Em tags `v*.*.*` ou dispatch manual, empacota e publica um OCI artifact no GHCR via ORAS.

## Estrutura

```
contracts/
  services/
    social-care/
      model/schemas/       # ~54 schemas YAML canônicos do domínio
      openapi/openapi.yaml # Contrato síncrono (OpenAPI 3.1)
      asyncapi/asyncapi.yaml # Contrato assíncrono (AsyncAPI 3.1, NATS)
  shared/
    schemas/               # Error.yaml, Pagination.yaml — schemas reutilizáveis
    validation-rules/      # Regras de validação por bounded context (kernel, registry, assessment, care, protection, analytics, cross-validations)
```

## Convenções

- **Modelo canônico**: cada serviço tem schemas em `model/schemas/`. OpenAPI e AsyncAPI referenciam esses schemas via `$ref` — nunca duplicar definições.
- **Schemas compartilhados**: `shared/schemas/` contém tipos reutilizáveis entre serviços (Error, Pagination).
- **Validation rules**: `shared/validation-rules/` define regras de validação por bounded context (kernel VOs, registry, assessment, care, protection) e cross-validations inter-campo. Prefixados com `_` para arquivos auxiliares (ex: `_custom-validators.yaml`).
- **operationId**: deve ser explícito e action-oriented no OpenAPI.
- **Eventos async**: nomes descrevem fatos concluídos (ex: `patient.created`, `family-member.added`). Todos carregam metadata (eventId, occurredAt, schemaVersion) e actorId.
- **Versionamento**: `info.version` nos contratos deve acompanhar a estratégia de release. Bundle publicado com tags semânticas `vMAJOR.MINOR.PATCH`.
- **Header X-Actor-Id**: obrigatório em todos os endpoints de mutação.

## Como Adicionar um Novo Serviço

1. Criar `services/<nome>/model/schemas/`
2. Criar `services/<nome>/openapi/openapi.yaml`
3. Criar `services/<nome>/asyncapi/asyncapi.yaml`
4. Referenciar schemas de `model/schemas/` via `$ref` em ambas as specs
5. Reutilizar schemas de `shared/schemas/` quando aplicável

## Bounded Contexts (social-care)

Os contratos e validation rules estão organizados em 5 contextos:
- **Kernel** — VOs cross-cutting: CPF, NIS, CEP, RG, Address, PersonId, ProfessionalId, LookupId, TimeStamp
- **Registry** — Cadastro de pacientes, membros familiares, identidade social
- **Assessment** — Avaliações (habitação, socioeconômica, educação, saúde, comunidade, resumo)
- **Care** — Atendimentos e informações de ingresso
- **Protection** — Encaminhamentos, violações de direitos, histórico de acolhimento
