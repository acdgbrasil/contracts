# Guia Completo para IA — Leitura de Contratos e Implementação de Testes

> **Objetivo**: Este documento é a referência definitiva para que qualquer IA (ou desenvolvedor) leia os contratos da ACDG e implemente testes unitários que validem **ponto a ponto** tudo que está definido nos contratos: schemas, endpoints, eventos, validações, invariantes e analytics.

---

## 1. VISÃO GERAL DO REPOSITÓRIO

```
contracts/
├── services/social-care/
│   ├── openapi/openapi.yaml          # Contrato síncrono (OpenAPI 3.1)
│   ├── asyncapi/asyncapi.yaml        # Contrato assíncrono (AsyncAPI 3.1, NATS)
│   └── model/schemas/                # 50 schemas YAML canônicos
├── shared/
│   ├── schemas/                      # Error.yaml, Pagination.yaml
│   └── validation-rules/            # 7 YAMLs com regras de validação
│       ├── kernel.yaml              # VOs cross-cutting (CPF, NIS, CEP, RG, Address)
│       ├── registry.yaml            # Patient, FamilyMember, PersonalData
│       ├── assessment.yaml          # Housing, Health, Education, SocioEconomic
│       ├── care.yaml                # Appointment, Diagnosis, ICDCode, IngressInfo
│       ├── protection.yaml          # Referral, Violation, PlacementHistory
│       ├── analytics.yaml           # Indicadores calculados
│       ├── cross-validations.yaml   # Regras inter-campo e metadata-driven
│       └── _custom-validators.yaml  # Algoritmos nativos (CPF checksum, etc.)
```

### Como os arquivos se conectam

```
OpenAPI/AsyncAPI ──$ref──▶ model/schemas/*.yaml ──$ref──▶ (entre si)
                                    ▲
shared/schemas/ ◀──$ref─────────────┘
shared/validation-rules/ ── define regras para cada schema
```

---

## 2. ORDEM DE LEITURA (OBRIGATÓRIA)

Para entender o domínio antes de testar, leia nesta ordem:

1. **`shared/validation-rules/kernel.yaml`** — VOs primitivos (CPF, NIS, CEP, etc.)
2. **`shared/validation-rules/registry.yaml`** — Cadastro de pacientes
3. **`shared/validation-rules/assessment.yaml`** — Avaliações sociais
4. **`shared/validation-rules/care.yaml`** — Atendimentos
5. **`shared/validation-rules/protection.yaml`** — Proteção e encaminhamentos
6. **`shared/validation-rules/analytics.yaml`** — Indicadores calculados
7. **`shared/validation-rules/cross-validations.yaml`** — Regras inter-campo
8. **`services/social-care/openapi/openapi.yaml`** — Endpoints HTTP
9. **`services/social-care/asyncapi/asyncapi.yaml`** — Eventos assíncronos
10. **`services/social-care/model/schemas/`** — Schemas individuais (sob demanda)

---

## 3. MAPA COMPLETO DE ENDPOINTS (OpenAPI)

### 3.1 Health (sem autenticação)

| Método | Path | operationId | Resposta | O que testar |
|--------|------|-------------|----------|-------------|
| GET | `/health` | healthCheck | `{ status: "ok" }` | Retorna 200 sempre |
| GET | `/ready` | readinessCheck | `ReadinessResponse` | 200 se DB ok, 503 se DB falha |

### 3.2 Registry (JWT + X-Actor-Id)

| Método | Path | operationId | Body/Params | Resposta | O que testar |
|--------|------|-------------|-------------|----------|-------------|
| POST | `/patients` | registerPatient | `RegisterPatientRequest` | 201 + `IdResponse` | Criação completa, todos os campos obrigatórios, rejeição de campos inválidos |
| GET | `/patients/{patientId}` | getPatientById | patientId (UUID) | `PatientResponse` | Projeção completa do agregado, 404 para ID inexistente |
| GET | `/patients/by-person/{personId}` | getPatientByPersonId | personId (UUID) | `PatientResponse` | Busca por personId, 404 se não existe |
| POST | `/patients/{patientId}/family-members` | addFamilyMember | `AddFamilyMemberRequest` | 204 | Adicionar membro, invariantes de duplicação e PR |
| DELETE | `/patients/{patientId}/family-members/{memberId}` | removeFamilyMember | patientId + memberId | 204 | Remover membro existente, 404/422 se não existe |
| PUT | `/patients/{patientId}/primary-caregiver` | assignPrimaryCaregiver | `AssignPrimaryCaregiverRequest` | 204 | Atribuir cuidador, membro deve existir |
| PUT | `/patients/{patientId}/social-identity` | updateSocialIdentity | `UpdateSocialIdentityRequest` | 204 | Tipo "OUTRAS" exige descrição |

### 3.3 Audit

| Método | Path | operationId | Params | Resposta | O que testar |
|--------|------|-------------|--------|----------|-------------|
| GET | `/patients/{patientId}/audit-trail` | getAuditTrail | eventType (opcional) | `AuditTrailEntryResponse[]` | Lista eventos, filtro por tipo |

### 3.4 Assessment (todos PUT, JWT + X-Actor-Id)

| Método | Path | operationId | Body Schema | O que testar |
|--------|------|-------------|-------------|-------------|
| PUT | `/patients/{patientId}/housing-condition` | updateHousingCondition | `HousingCondition` | Enums, contagens ≥ 0, bedrooms ≤ rooms |
| PUT | `/patients/{patientId}/socioeconomic-situation` | updateSocioEconomicSituation | `SocioEconomicSituation` | Renda ≥ 0, perCapita ≤ total, flag↔benefits |
| PUT | `/patients/{patientId}/work-and-income` | updateWorkAndIncome | `UpdateWorkAndIncomeRequest` | Rendas individuais ≥ 0 |
| PUT | `/patients/{patientId}/educational-status` | updateEducationalStatus | `UpdateEducationalStatusRequest` | Perfis e ocorrências |
| PUT | `/patients/{patientId}/health-status` | updateHealthStatus | `UpdateHealthStatusRequest` | Deficiências, gestantes (cross: sexo), cuidados |
| PUT | `/patients/{patientId}/community-support-network` | updateCommunitySupportNetwork | `UpdateCommunitySupportNetworkRequest` | familyConflicts max 300 chars |
| PUT | `/patients/{patientId}/social-health-summary` | updateSocialHealthSummary | `UpdateSocialHealthSummaryRequest` | functionalDependencies sem itens vazios |

### 3.5 Care (JWT + X-Actor-Id)

| Método | Path | operationId | Body Schema | O que testar |
|--------|------|-------------|-------------|-------------|
| POST | `/patients/{patientId}/appointments` | registerAppointment | `RegisterAppointmentRequest` | Data não futura, ≥1 narrative, limites de texto |
| PUT | `/patients/{patientId}/intake-info` | registerIntakeInfo | `RegisterIntakeInfoRequest` | serviceReason obrigatório |

### 3.6 Protection (JWT + X-Actor-Id)

| Método | Path | operationId | Body Schema | O que testar |
|--------|------|-------------|-------------|-------------|
| PUT | `/patients/{patientId}/placement-history` | updatePlacementHistory | `UpdatePlacementHistoryRequest` | endDate ≥ startDate, cross-validations |
| POST | `/patients/{patientId}/violation-reports` | reportRightsViolation | `ReportRightsViolationRequest` | incidentDate ≤ reportDate, descrição obrigatória |
| POST | `/patients/{patientId}/referrals` | createReferral | `CreateReferralRequest` | Data não futura, motivo obrigatório |

### 3.7 Lookup

| Método | Path | operationId | Params | O que testar |
|--------|------|-------------|--------|-------------|
| GET | `/dominios/{tableName}` | listLookupItems | tableName (enum 13 valores) | Retorna array de `{id, codigo, descricao}`, rejeita tableName inválido |

**Tabelas de domínio válidas:** `dominio_tipo_identidade`, `dominio_parentesco`, `dominio_condicao_ocupacao`, `dominio_escolaridade`, `dominio_efeito_condicionalidade`, `dominio_tipo_deficiencia`, `dominio_programa_social`, `dominio_tipo_ingresso`, `dominio_tipo_beneficio`, `dominio_tipo_violacao`, `dominio_servico_vinculo`, `dominio_tipo_medida`, `dominio_unidade_realizacao`

### 3.8 Erros Padronizados (todas as rotas)

| HTTP Status | Quando | Schema |
|-------------|--------|--------|
| 400 | Request malformado | `ErrorResponse` |
| 404 | Recurso não encontrado | `ErrorResponse` |
| 409 | Conflito (duplicação) | `ErrorResponse` |
| 422 | Regra de negócio violada | `ErrorResponse` |
| 500 | Erro interno | `ErrorResponse` |

**Estrutura do ErrorResponse:**
```yaml
success: false
error:
  id: UUID
  code: "PAT-001"      # código estável por tipo de erro
  message: string
  bc: "registry"        # bounded context
  module: "patient"
  kind: "INVARIANT_VIOLATION"
  observability:
    category: DOMAIN_RULE_VIOLATION | CONFLICT | ...
    severity: ERROR | WARNING | ...
    fingerprint: [string]
    tags: {key: value}
  http: 422
details: [{ field, message }]  # opcional
```

### 3.9 Headers Obrigatórios

| Header | Onde | Formato | O que testar |
|--------|------|---------|-------------|
| `Authorization` | Todas (exceto health/ready) | `Bearer <JWT>` | 401 sem token, 403 sem role |
| `X-Actor-Id` | Todos os endpoints de mutação (POST/PUT/DELETE) | UUID | 400 sem header |

---

## 4. MAPA COMPLETO DE EVENTOS (AsyncAPI)

### 4.1 Registry — Lifecycle

| Canal NATS | Mensagem | Payload | Dados chave |
|------------|----------|---------|-------------|
| `social-care.patient.created` | PatientCreated | `PatientCreatedEventPayload` | patientId, personId |

### 4.2 Registry — Family

| Canal NATS | Mensagem | Payload | Dados chave |
|------------|----------|---------|-------------|
| `social-care.family-member.added` | FamilyMemberAdded | `FamilyMemberAddedEventPayload` | patientId, memberId, relationship |
| `social-care.family-member.removed` | FamilyMemberRemoved | `FamilyMemberRemovedEventPayload` | patientId, memberId |
| `social-care.primary-caregiver.assigned` | PrimaryCaregiverAssigned | `PrimaryCaregiverAssignedEventPayload` | patientId, caregiverId |
| `social-care.social-identity.updated` | SocialIdentityUpdated | `AssessmentUpdatedEventPayload` | patientId, before, after |

### 4.3 Assessment (todos usam AssessmentUpdatedEventPayload)

| Canal NATS | Mensagem |
|------------|----------|
| `social-care.housing-condition.updated` | HousingConditionUpdated |
| `social-care.socioeconomic-situation.updated` | SocioEconomicSituationUpdated |
| `social-care.work-and-income.updated` | WorkAndIncomeUpdated |
| `social-care.educational-status.updated` | EducationalStatusUpdated |
| `social-care.health-status.updated` | HealthStatusUpdated |
| `social-care.community-support-network.updated` | CommunitySupportNetworkUpdated |
| `social-care.social-health-summary.updated` | SocialHealthSummaryUpdated |
| `social-care.intake-info.updated` | IntakeInfoUpdated |

### 4.4 Care

| Canal NATS | Mensagem | Payload | Dados chave |
|------------|----------|---------|-------------|
| `social-care.appointment.registered` | AppointmentRegistered | `SocialCareAppointmentRegisteredEventPayload` | patientId, appointmentId, professionalInChargeId, type |
| `social-care.placement-history.updated` | PlacementHistoryUpdated | `AssessmentUpdatedEventPayload` | patientId, before, after |

### 4.5 Protection

| Canal NATS | Mensagem | Payload | Dados chave |
|------------|----------|---------|-------------|
| `social-care.rights-violation.reported` | RightsViolationReported | `RightsViolationReportedEventPayload` | patientId, reportId, victimId, violationType |
| `social-care.referral.created` | ReferralCreated | `ReferralCreatedEventPayload` | patientId, referralId, referredPersonId, destinationService, status |

### Estrutura de todos os eventos

```yaml
metadata:
  eventId: UUID          # identificador único do evento
  occurredAt: IsoDateTime # quando aconteceu
  schemaVersion: "1.0.0"  # versão do schema do payload
data:
  # campos específicos do evento
```

---

## 5. SCHEMAS — CATÁLOGO COMPLETO

### 5.1 Tipos Primitivos

| Schema | Tipo | Formato | Validação |
|--------|------|---------|-----------|
| `Uuid.yaml` | string | uuid | UUID qualquer versão |
| `IsoDateTime.yaml` | string | date-time | ISO 8601 com timezone |
| `IsoDate.yaml` | string | date | YYYY-MM-DD |
| `SemanticVersion.yaml` | string | — | MAJOR.MINOR.PATCH |

### 5.2 Enums

| Schema | Valores | Onde é usado |
|--------|---------|-------------|
| `HousingConditionType.yaml` | OWNED, RENTED, CEDED, SQUATTED | HousingCondition.type |
| `WallMaterial.yaml` | MASONRY, FINISHED_WOOD, MAKESHIFT_MATERIALS | HousingCondition.wallMaterial |
| `WaterSupplyType.yaml` | PUBLIC_NETWORK, WELL_OR_SPRING, RAINWATER_HARVEST, WATER_TRUCK, OTHER | HousingCondition.waterSupply |
| `ElectricityAccess.yaml` | METERED_CONNECTION, IRREGULAR_CONNECTION, NO_ACCESS | HousingCondition.electricityAccess |
| `SewageDisposalMethod.yaml` | PUBLIC_SEWER, SEPTIC_TANK, RUDIMENTARY_PIT, OPEN_SEWAGE, NO_BATHROOM | HousingCondition.sewageDisposal |
| `WasteCollectionType.yaml` | DIRECT_COLLECTION, INDIRECT_COLLECTION, NO_COLLECTION | HousingCondition.wasteCollection |
| `AccessibilityLevel.yaml` | FULLY_ACCESSIBLE, PARTIALLY_ACCESSIBLE, NOT_ACCESSIBLE | HousingCondition.accessibilityLevel |
| `SocialCareAppointmentType.yaml` | HOME_VISIT, OFFICE_APPOINTMENT, PHONE_CALL, MULTIDISCIPLINARY, OTHER | Appointment.type |
| `ViolationType.yaml` | NEGLECT, PSYCHOLOGICAL_VIOLENCE, PHYSICAL_VIOLENCE, SEXUAL_ABUSE, SEXUAL_EXPLOITATION, CHILD_LABOR, FINANCIAL_EXPLOITATION, DISCRIMINATION, OTHER | RightsViolationReport.violationType |
| `ReferralDestinationService.yaml` | CRAS, CREAS, HEALTH_CARE, EDUCATION, LEGAL, OTHER | Referral.destinationService |
| `ReferralStatus.yaml` | PENDING, COMPLETED, CANCELLED | Referral.status (state machine) |

### 5.3 Request DTOs

| Schema | Campos obrigatórios | Referências |
|--------|---------------------|-------------|
| `RegisterPatientRequest` | personId, initialDiagnoses (≥1), prRelationshipId, personalData, civilDocuments, address, socialIdentity | InitialDiagnosis, IsoDate, Uuid |
| `AddFamilyMemberRequest` | memberPersonId, relationship, isResiding, isCaregiver, hasDisability, birthDate, prRelationshipId | Uuid, IsoDate |
| `AssignPrimaryCaregiverRequest` | memberPersonId | Uuid |
| `UpdateSocialIdentityRequest` | typeId | Uuid |
| `HousingCondition` | type, wallMaterial, numberOfRooms, numberOfBedrooms, numberOfBathrooms, waterSupply, hasPipedWater, electricityAccess, sewageDisposal, wasteCollection, accessibilityLevel, isInGeographicRiskArea, hasDifficultAccess, isInSocialConflictArea, hasDiagnosticObservations | Enums de housing |
| `SocioEconomicSituation` | totalFamilyIncome, incomePerCapita, receivesSocialBenefit, socialBenefits, mainSourceOfIncome, hasUnemployed | SocialBenefit |
| `UpdateWorkAndIncomeRequest` | individualIncomes, socialBenefits, hasRetiredMembers | SocialBenefit |
| `UpdateEducationalStatusRequest` | memberProfiles, programOccurrences | — |
| `UpdateHealthStatusRequest` | deficiencies, gestatingMembers, constantCareNeeds, foodInsecurity | Uuid |
| `UpdateCommunitySupportNetworkRequest` | hasRelativeSupport, hasNeighborSupport, familyConflicts, patientParticipatesInGroups, familyParticipatesInGroups, patientHasAccessToLeisure, facesDiscrimination | — |
| `UpdateSocialHealthSummaryRequest` | requiresConstantCare, hasMobilityImpairment, functionalDependencies, hasRelevantDrugTherapy | — |
| `RegisterAppointmentRequest` | professionalId, summary, actionPlan, date, type | Uuid, IsoDateTime, AppointmentType |
| `RegisterIntakeInfoRequest` | ingressTypeId, originName, originContact, serviceReason, linkedSocialPrograms | Uuid |
| `UpdatePlacementHistoryRequest` | registries, collectiveSituations, separationChecklist | IsoDateTime |
| `ReportRightsViolationRequest` | victimId, violationType, descriptionOfFact, violationTypeId, reportDate | Uuid, ViolationType, IsoDateTime |
| `CreateReferralRequest` | referredPersonId, destinationService, reason, date, professionalId | Uuid, DestinationService, IsoDateTime |

### 5.4 Response Schemas

| Schema | Campos | Onde aparece |
|--------|--------|-------------|
| `StandardResponse` | `{ data: T, meta: { timestamp } }` | Envelope de todas as respostas de leitura |
| `IdResponse` | `{ id: UUID }` | POST que cria recurso |
| `ErrorResponse` | `{ success: false, error: Error, details }` | Todas as respostas de erro |
| `ReadinessResponse` | `{ status: "ready" \| "unavailable" }` | GET /ready |
| `LookupItemResponse` | `{ id, codigo, descricao }` | GET /dominios/{table} |
| `AuditTrailEntryResponse` | `{ id, aggregateId, eventType, payload, occurredAt, recordedAt, actorId }` | GET audit-trail |
| `PatientResponse` | Projeção completa do agregado (ver seção 5.5) | GET patient |

### 5.5 PatientResponse — Projeção do Agregado

```yaml
patientId: UUID
personId: UUID
version: integer          # concorrência otimista
personalData:
  firstName, lastName, motherName, nationality, sex, socialName?, birthDate, phone?
civilDocuments:
  cpf?, nis?, rgDocument? { number, issuingState, issuingAgency, issueDate }
address:
  cep?, state, city, street?, neighborhood?, number?, complement?,
  residenceLocation (URBANO|RURAL), isShelter
socialIdentity:
  typeId, otherDescription?
familyMembers[]:
  personId, relationshipId, isPrimaryCaregiver, residesWithPatient,
  hasDisability, requiredDocuments[], birthDate
diagnoses[]:
  icdCode, date, description
housingCondition: (14 campos, ver HousingCondition)
socioEconomicSituation: (6 campos, ver SocioEconomicSituation)
workAndIncome:
  individualIncomes[] { memberId, occupationId?, hasWorkCard, monthlyAmount }
  socialBenefits[] { benefitName, amount, beneficiaryId }
  hasRetiredMembers
educationalStatus:
  memberProfiles[] { memberId, canReadWrite, attendsSchool, educationLevelId? }
  programOccurrences[] { date, effectId, isSuspensionRequested }
healthStatus:
  deficiencies[] { memberId, deficiencyTypeId }
  gestatingMembers[] { memberId }
  constantCareNeeds[] { memberId, description }
  foodInsecurity (NONE|MILD|MODERATE|SEVERE)
communitySupportNetwork: (7 booleans/text)
socialHealthSummary: (4 campos)
intakeInfo:
  ingressTypeId, originName?, originContact?, serviceReason, linkedSocialPrograms[]
appointments[]:
  id, date, professionalInChargeId, type, summary?, actionPlan?
referrals[]:
  id, date, requestingProfessionalId, referredPersonId, destinationService, reason, status
placementHistory:
  registries[] { memberId, startDate, endDate?, reason }
  collectiveSituations { homeLossReport, thirdPartyGuardReport }
  separationChecklist { adultInPrison, adolescentInInternment }
violationReports[]:
  id, reportDate, incidentDate?, victimId, violationType, descriptionOfFact, actionsTaken?
computedAnalytics:
  housing: { density, isOvercrowded }
  financial: { totalWorkIncome, perCapitaWorkIncome, totalGlobalIncome, perCapitaGlobalIncome }
  ageProfile: { bucket_0_6, bucket_7_14, ..., bucket_70_plus }
  educationalVulnerabilities: { notInSchool: {by_age_range}, illiteracy: {by_age_range} }
```

---

## 6. REGRAS DE VALIDAÇÃO — REFERÊNCIA COMPLETA

### 6.1 Kernel (VOs Cross-Cutting)

#### CPF
| Regra | Código | Detalhe |
|-------|--------|---------|
| Não vazio | CPF-001 | Após trim |
| Apenas dígitos (após strip de `.` e `-`) | CPF-002 | Regex: `^\d+$` |
| Exatamente 11 dígitos | CPF-003 | Após normalização |
| Sem dígitos repetidos | CPF-004 | `00000000000` a `99999999999` |
| Checksum válido | CPF-005 | Algoritmo MOD 11 (ver `_custom-validators.yaml`) |
| **Derivado**: fiscalRegion | — | Baseado no dígito na posição 8 |

#### NIS
| Regra | Código | Detalhe |
|-------|--------|---------|
| Não vazio | NIS-001 | Após strip de não-dígitos |
| Exatamente 11 dígitos | NIS-002 | — |

#### CEP
| Regra | Código | Detalhe |
|-------|--------|---------|
| Não vazio | CEP-001 | Após strip de `-` e espaços |
| Apenas dígitos | CEP-002 | — |
| Exatamente 8 dígitos | CEP-003 | — |
| Faixa de UF válida | CEP-004 | Primeiro dígito 0-9 mapeia para UFs |
| **Derivado**: distributionKind | — | Faixas especiais (00000-999 a 99999-999) |

#### RGDocument (objeto)
| Campo | Regra | Código |
|-------|-------|--------|
| number | Não vazio, formato 8 dígitos + check digit (0-9 ou X) | RG-001, RG-002 |
| issuingState | UF brasileira válida (27 valores) | RG-003 |
| issuingAgency | Não vazio | RG-004 |
| issueDate | Não no futuro | RG-005 |
| **Normalização**: number → strip `.` `-`, uppercase; state → uppercase; agency → trim, collapse spaces, uppercase |

#### Address (objeto)
| Campo | Regra | Código |
|-------|-------|--------|
| state | Obrigatório, UF válida | ADDR-001, ADDR-002 |
| city | Obrigatório | ADDR-003 |
| cep | Se fornecido, deve ser CEP válido | ADDR-004 |
| residenceLocation | Enum: URBANO, RURAL | ADDR-005 |
| isShelter | Boolean obrigatório | — |
| **Normalização**: todos os campos de texto → trim, collapse whitespace; state → uppercase |

#### IDs baseados em UUID
| Tipo | Regra |
|------|-------|
| PersonId, ProfessionalId, LookupId, PatientId, AppointmentId, ReferralId, ViolationReportId | Formato UUID válido, auto-gerado se não fornecido, normaliza para lowercase |

#### TimeStamp
| Regra | Detalhe |
|-------|---------|
| Formato ISO 8601 com timezone | `yyyy-MM-dd'T'HH:mm:ss.SSSZ` |
| Métodos: `isSameDay(other)`, `years_at(reference)`, `toISOString()` |

### 6.2 Registry

#### PersonalData
| Campo | Regra | Código |
|-------|-------|--------|
| firstName | Obrigatório, não vazio após trim | PD-001 |
| lastName | Obrigatório, não vazio | PD-002 |
| motherName | Obrigatório, não vazio | PD-003 |
| nationality | Obrigatório, não vazio | PD-004 |
| sex | Obrigatório, enum MASCULINO/FEMININO/OUTRO | PD-005 |
| birthDate | Obrigatório, não no futuro | PD-006 |
| socialName | Opcional, `null_if_empty: true` |
| phone | Opcional, `null_if_empty: true` |
| **Normalização**: todos os nomes → trim, collapse whitespace |

#### CivilDocuments
| Regra | Código | Detalhe |
|-------|--------|---------|
| Pelo menos 1 documento (cpf, nis ou rgDocument) | CD-001 | Todos podem ser nil, mas não todos ao mesmo tempo |

#### SocialIdentity
| Regra | Código | Detalhe |
|-------|--------|---------|
| typeId obrigatório (referencia `dominio_tipo_identidade`) | SI-001 | — |
| Se tipo é "OUTRAS", description obrigatória | SI-002 | Não vazio após trim |

#### FamilyMember
| Campo/Regra | Código |
|-------------|--------|
| personId obrigatório (UUID) | FM-001 |
| relationshipId referencia `dominio_parentesco` | FM-002 |
| birthDate obrigatório | FM-003 |

#### Patient (Agregado) — Invariantes
| Invariante | Código | Detalhe |
|------------|--------|---------|
| Pelo menos 1 diagnóstico inicial | PAT-001 | `initialDiagnoses.count ≥ 1` |
| Exatamente 1 Pessoa de Referência (PR) | PAT-002 | Membro com `prRelationshipId` |
| Sem personId duplicado entre membros | PAT-003 | — |
| Não pode adicionar segundo PR | PAT-004 | — |
| Membro deve existir para remoção | PAT-005 | — |
| Alvo de referral deve estar no boundary | PAT-006 | Paciente ou membro familiar |
| Vítima de violação deve estar no boundary | PAT-007 | Paciente ou membro familiar |
| adolescentInInternment → membro 12-17 anos | PAT-008 | Cross-validation com idade |
| thirdPartyGuardReport → membro 0-17 anos | PAT-009 | Cross-validation com idade |

### 6.3 Assessment

#### HousingCondition
| Regra | Código |
|-------|--------|
| numberOfRooms ≥ 0 | HC-001 |
| numberOfBedrooms ≥ 0 | HC-002 |
| numberOfBathrooms ≥ 0 | HC-003 |
| numberOfBedrooms ≤ numberOfRooms | HC-004 |
| Todos os campos enum devem ser valores válidos | HC-005 |

#### SocioEconomicSituation
| Regra | Código |
|-------|--------|
| totalFamilyIncome ≥ 0 | SES-001 |
| incomePerCapita ≥ 0 | SES-002 |
| incomePerCapita ≤ totalFamilyIncome | SES-003 |
| mainSourceOfIncome não vazio | SES-004 |
| `receivesSocialBenefit == true` → socialBenefits não vazio | SES-005 |
| `receivesSocialBenefit == false` → socialBenefits vazio | SES-006 |

#### SocialBenefit
| Regra | Código |
|-------|--------|
| benefitName não vazio | SB-001 |
| amount > 0 | SB-002 |
| Sem nomes duplicados na coleção | SB-003 |

#### WorkAndIncome
| Regra | Código |
|-------|--------|
| monthlyAmount ≥ 0 para cada individualIncome | WI-001 |

#### CommunitySupportNetwork
| Regra | Código |
|-------|--------|
| familyConflicts: se fornecido, não apenas whitespace | CSN-001 |
| familyConflicts: máximo 300 caracteres | CSN-002 |

#### SocialHealthSummary
| Regra | Código |
|-------|--------|
| functionalDependencies: itens não podem ser vazios | SHS-001 |
| functionalDependencies: deduplicação automática | SHS-002 |

### 6.4 Care

#### ICDCode
| Regra | Código |
|-------|--------|
| Não vazio | ICD-001 |
| Pattern: `^[A-TV-Z][0-9][0-9AB](\.[0-9A-TV-Z]{1,4})?$` | ICD-002 |
| Auto-dot: insere `.` antes do 4o caractere se ausente | ICD-003 |
| Normaliza para uppercase | ICD-004 |

#### Diagnosis
| Regra | Código |
|-------|--------|
| icdCode válido | DIAG-001 |
| date não no futuro | DIAG-002 |
| description não vazia (min 10, max 2000 chars) | DIAG-003 |

#### IngressInfo
| Regra | Código |
|-------|--------|
| ingressTypeId referencia `dominio_tipo_ingresso` | ING-001 |
| serviceReason não vazio | ING-002 |

#### SocialCareAppointment
| Regra | Código |
|-------|--------|
| date não no futuro | SCA-001 |
| Pelo menos um de summary ou actionPlan preenchido | SCA-002 |
| summary: max 500 caracteres | SCA-003 |
| actionPlan: max 2000 caracteres | SCA-004 |
| type: enum válido | SCA-005 |

### 6.5 Protection

#### Referral
| Regra | Código |
|-------|--------|
| date não no futuro | REF-001 |
| reason não vazio | REF-002 |
| destinationService: enum válido | REF-003 |
| Status inicia como PENDING | REF-004 |
| **State machine**: PENDING → COMPLETED \| CANCELLED (terminais) | REF-005 |

#### RightsViolationReport
| Regra | Código |
|-------|--------|
| reportDate não no futuro | VIO-001 |
| incidentDate ≤ reportDate (se fornecido) | VIO-002 |
| descriptionOfFact não vazia | VIO-003 |
| violationType: enum válido | VIO-004 |

#### PlacementHistory
| Regra | Código |
|-------|--------|
| endDate ≥ startDate (se endDate fornecido) | PH-001 |
| endDate nil = acolhimento em andamento | PH-002 |

### 6.6 Cross-Validations (Inter-Campo)

| ID | Regra | Campos envolvidos |
|----|-------|-------------------|
| cv_pregnancy_rejects_male | Gestantes não podem ser do sexo MASCULINO | healthStatus.gestatingMembers × familyMembers.personalData.sex |
| cv_placement_date_chronology | endDate ≥ startDate | placementHistory.registries |
| cv_third_party_guard_requires_minor | thirdPartyGuardReport → membro 0-17 | collectiveSituations × familyMembers.birthDate |
| cv_adolescent_internment_requires_12_17 | adolescentInInternment → membro 12-17 | separationChecklist × familyMembers.birthDate |

### 6.7 Metadata-Driven Validations

| ID | Regra | Lookup table |
|----|-------|-------------|
| mv_benefit_requires_birth_certificate | Se flag `exige_registro_nascimento` ativa no tipo de benefício | dominio_tipo_beneficio |
| mv_benefit_requires_deceased_cpf | Se flag `exige_cpf_falecido` ativa no tipo de benefício | dominio_tipo_beneficio |
| mv_violation_requires_description | Se flag `exige_descricao` ativa no tipo de violação | dominio_tipo_violacao |

---

## 7. ANALYTICS — INDICADORES CALCULADOS

### 7.1 Housing Analytics
```
density = max(memberCount, 1) / max(bedroomCount, 1)
isOvercrowded = density > 3.0
```

### 7.2 Financial Analytics
```
totalWorkIncome = sum(individualIncomes[].monthlyAmount)
perCapitaWorkIncome = totalWorkIncome / max(memberCount, 1)
totalGlobalIncome = totalWorkIncome + sum(socialBenefits[].amount)
perCapitaGlobalIncome = totalGlobalIncome / max(memberCount, 1)
```

### 7.3 Age Profile (FamilyAnalytics)
```
Buckets: 0-6, 7-14, 15-17, 18-29, 30-59, 60-64, 65-69, 70+
Cálculo: years_at(referenceDate) para cada membro com birthDate
```

### 7.4 Educational Vulnerabilities
```
notInSchool:
  - Faixa 0-5: membros que não frequentam escola
  - Faixa 6-14: membros que não frequentam escola (obrigatório)
  - Faixa 15-17: membros que não frequentam escola

illiteracy:
  - Faixa 10-17: membros que não sabem ler/escrever
  - Faixa 18-59: membros que não sabem ler/escrever
  - Faixa 60+: membros que não sabem ler/escrever
```

---

## 8. ESTRATÉGIA DE TESTES — O QUE TESTAR E COMO

### 8.1 Camadas de Teste

```
┌─────────────────────────────────────────────────────────────┐
│ CAMADA 1: Value Objects (Domain)                            │
│ → Validação, normalização, rejeição, igualdade              │
├─────────────────────────────────────────────────────────────┤
│ CAMADA 2: Agregados e Invariantes (Domain)                  │
│ → Invariantes do Patient, state machine do Referral         │
├─────────────────────────────────────────────────────────────┤
│ CAMADA 3: Analytics Services (Domain)                       │
│ → Cálculos de indicadores, edge cases numéricos             │
├─────────────────────────────────────────────────────────────┤
│ CAMADA 4: Use Cases (Application)                           │
│ → Comandos CQRS, orquestração, erros de aplicação          │
├─────────────────────────────────────────────────────────────┤
│ CAMADA 5: Cross-Validations (IO/HTTP)                       │
│ → Validações inter-campo, metadata-driven                   │
├─────────────────────────────────────────────────────────────┤
│ CAMADA 6: DTOs e Contratos HTTP (IO/HTTP)                   │
│ → Serialização/deserialização, campos obrigatórios,         │
│   headers, status codes                                     │
├─────────────────────────────────────────────────────────────┤
│ CAMADA 7: Eventos (IO/EventBus)                             │
│ → Payload correto, metadata presente, canal correto         │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 CAMADA 1 — Value Objects

**Para CADA Value Object**, implementar:

| Categoria | O que testar | Exemplo |
|-----------|-------------|---------|
| Happy path | Criação com valores válidos | `CPF("529.982.247-25")` → sucesso |
| Normalização | Trim, collapse, uppercase, strip chars | `CPF("  529.982.247-25  ")` → `"52998224725"` |
| Campo vazio | String vazia, apenas espaços, nil | `CPF("")` → erro |
| Formato inválido | Comprimento errado, chars inválidos | `CPF("123")` → erro |
| Regra de negócio | Checksum, ranges, cross-field | `CPF("11111111111")` → erro |
| Boundary values | Limites exatos | `familyConflicts(300 chars)` → ok, `(301)` → erro |
| Igualdade | Equatable/Hashable | `CPF("529.982.247-25") == CPF("52998224725")` |

**VOs a testar (30+):**
CPF, NIS, CEP, PersonId, ProfessionalId, LookupId, PatientId, AppointmentId, ReferralId, ViolationReportId, RGDocument, Address, TimeStamp, PersonalData, CivilDocuments, SocialIdentity, HousingCondition, SocioEconomicSituation, SocialBenefit, SocialBenefitsCollection, WorkIncomeVO, CommunitySupportNetwork, SocialHealthSummary, EducationalStatus, HealthStatus, ICDCode, Diagnosis, IngressInfo, SocialCareAppointment, Referral, RightsViolationReport, PlacementHistory

### 8.3 CAMADA 2 — Agregados e Invariantes

**Patient Aggregate — testar cada invariante:**

| Teste | Invariante | Cenário |
|-------|-----------|---------|
| Criação com ≥1 diagnóstico | PAT-001 | Happy path + rejeição sem diagnósticos |
| Exatamente 1 PR | PAT-002 | Criação válida + rejeição sem PR |
| Sem membros duplicados | PAT-003 | Adicionar 2 membros com mesmo personId → erro |
| Sem segundo PR | PAT-004 | Adicionar membro como PR quando já existe → erro |
| Membro deve existir para remoção | PAT-005 | Remover membro inexistente → erro |
| Alvo de referral no boundary | PAT-006 | Referral para personId fora do boundary → erro |
| Vítima de violação no boundary | PAT-007 | Violação com victimId fora do boundary → erro |

**Referral State Machine:**

| Teste | Transição |
|-------|-----------|
| Inicializa como PENDING | `Referral.new()` → `.pending` |
| PENDING → COMPLETED | `ref.complete()` → `.completed` |
| PENDING → CANCELLED | `ref.cancel()` → `.cancelled` |
| COMPLETED → * (rejeitar) | `ref.complete()` ou `ref.cancel()` → erro |
| CANCELLED → * (rejeitar) | `ref.complete()` ou `ref.cancel()` → erro |

### 8.4 CAMADA 3 — Analytics Services

| Service | Testes |
|---------|--------|
| HousingAnalyticsService | density(6, 2) = 3.0; density(0, 0) = 1.0; isOvercrowded(7, 2) = true; isOvercrowded(6, 2) = false |
| FinancialAnalyticsService | totalWorkIncome, perCapita, totalGlobal com valores conhecidos; memberCount = 0 → usa 1 |
| FamilyAnalyticsService | age buckets com datas de nascimento conhecidas; membro sem birthDate |
| EducationAnalyticsService | notInSchool por faixa etária; illiteracy por faixa etária; membro na fronteira de faixa |

### 8.5 CAMADA 4 — Use Cases

**Para CADA use case (17 total), testar:**

| Categoria | O que testar |
|-----------|-------------|
| Happy path | Comando válido → sucesso + evento emitido |
| Validação de comando | Campos obrigatórios ausentes → erro |
| Paciente não encontrado | patientId inexistente → erro NOT_FOUND |
| Invariante violada | Regra de negócio do agregado → erro UNPROCESSABLE |
| Concorrência | Versão incorreta → erro CONFLICT (409) |
| Evento emitido | Verificar que o EventBus recebeu o evento correto com metadata |

**Lista dos 17 use cases:**

| Bounded Context | Use Case | Comando |
|----------------|----------|---------|
| Registry | RegisterPatient | RegisterPatientCommand |
| Registry | AddFamilyMember | AddFamilyMemberCommand |
| Registry | RemoveFamilyMember | RemoveFamilyMemberCommand |
| Registry | AssignPrimaryCaregiver | AssignPrimaryCaregiverCommand |
| Registry | UpdateSocialIdentity | UpdateSocialIdentityCommand |
| Assessment | UpdateHousingCondition | UpdateHousingConditionCommand |
| Assessment | UpdateHealthStatus | UpdateHealthStatusCommand |
| Assessment | UpdateEducationalStatus | UpdateEducationalStatusCommand |
| Assessment | UpdateSocioEconomicSituation | UpdateSocioEconomicSituationCommand |
| Assessment | UpdateWorkAndIncome | UpdateWorkAndIncomeCommand |
| Assessment | UpdateSocialBenefits | UpdateSocialBenefitsCommand |
| Assessment | UpdateCommunitySupportNetwork | UpdateCommunitySupportNetworkCommand |
| Assessment | UpdateSocialHealthSummary | UpdateSocialHealthSummaryCommand |
| Care | RegisterAppointment | RegisterAppointmentCommand |
| Care | RegisterIntakeInfo | RegisterIntakeInfoCommand |
| Protection | CreateReferral | CreateReferralCommand |
| Protection | ReportRightsViolation | ReportRightsViolationCommand |
| Protection | UpdatePlacementHistory | UpdatePlacementHistoryCommand |

### 8.6 CAMADA 5 — Cross-Validations

| Teste | Regra | Cenário |
|-------|-------|---------|
| Gestante do sexo masculino → erro | cv_pregnancy_rejects_male | HealthStatus com gestatingMember cujo sex é MASCULINO |
| Gestante do sexo feminino → ok | cv_pregnancy_rejects_male | HealthStatus com gestatingMember cujo sex é FEMININO |
| Gestante do sexo outro → ok | cv_pregnancy_rejects_male | HealthStatus com gestatingMember cujo sex é OUTRO |
| thirdPartyGuardReport sem menor → erro | cv_third_party_guard_requires_minor | collectiveSituations.thirdPartyGuardReport=true sem membro 0-17 |
| thirdPartyGuardReport com menor → ok | cv_third_party_guard_requires_minor | collectiveSituations.thirdPartyGuardReport=true com membro 10 anos |
| adolescentInInternment sem 12-17 → erro | cv_adolescent_internment_requires_12_17 | separationChecklist.adolescentInInternment=true sem membro 12-17 |
| adolescentInInternment com 14 anos → ok | cv_adolescent_internment_requires_12_17 | separationChecklist.adolescentInInternment=true com membro 14 anos |
| Benefício exige certidão → valida | mv_benefit_requires_birth_certificate | Lookup flag ativa |
| Violação exige descrição → valida | mv_violation_requires_description | Lookup flag ativa |

### 8.7 CAMADA 6 — DTOs e Contratos HTTP

**Para CADA endpoint:**

| Categoria | O que testar |
|-----------|-------------|
| Request DTO válido | JSON com todos os campos obrigatórios → desserializa corretamente |
| Campo obrigatório ausente | Remover cada campo required → 400 ou 422 |
| Tipo errado | Enviar string onde esperava number → 400 |
| Enum inválido | Valor fora do enum → 422 |
| Response DTO | Verificar que a resposta contém todos os campos do schema |
| StandardResponse envelope | `{ data: ..., meta: { timestamp: IsoDateTime } }` |
| ErrorResponse formato | `{ success: false, error: { code, message, bc, ... } }` |
| Status codes | 201 para criação, 204 para update, 404 para não encontrado, 409 para conflito, 422 para regra violada |
| Header X-Actor-Id ausente | → 400 em endpoints de mutação |
| Header Authorization ausente | → 401 |
| JWT sem role adequada | → 403 |
| patientId formato inválido na URL | → 400 |

**Verificar conformidade de cada schema:**

Para cada Request DTO no OpenAPI, validar que TODOS os campos definidos no schema YAML são aceitos e que campos extras são ignorados ou rejeitados. Para cada Response DTO, validar que a resposta contém exatamente os campos definidos.

### 8.8 CAMADA 7 — Eventos

**Para CADA evento (15 total):**

| Categoria | O que testar |
|-----------|-------------|
| Metadata presente | eventId (UUID), occurredAt (IsoDateTime), schemaVersion |
| Payload correto | Todos os campos de `data` presentes e com tipos corretos |
| Canal correto | Evento publicado no canal NATS esperado |
| Idempotência | Mesmo comando não gera evento duplicado |
| Transactional Outbox | Evento persiste no outbox junto com a transação do agregado |

---

## 9. STACK DE TESTES (Swift)

```swift
import Testing                    // swift-testing (NÃO usar XCTest)
import Foundation
@testable import social_care_s   // módulo do serviço
```

### Convenções obrigatórias

| Item | Regra |
|------|-------|
| Framework | `swift-testing` (Swift 6.2) |
| Assertions | `#expect()` — NUNCA `XCTAssert` |
| Suítes | `struct` com `@Suite` — NUNCA classes |
| Async | `async/await` em testes que tocam actors/handlers |
| `@Suite` | Descrição em português: `"CPF - Validações"` |
| `@Test` | Descrição em português: `"Deve rejeitar CPF com todos dígitos iguais"` |
| Nome do método | camelCase em inglês: `rejectAllSameDigits()` |
| Erros sync | `#expect(throws: ErrorType.case) { try VO(...) }` |
| Erros async | `await #expect(throws: ErrorType.self) { try await handler.handle(...) }` |

### Estrutura de arquivo

```swift
import Testing
import Foundation
@testable import social_care_s

@Suite("NomeDoVO - Validações")
struct NomeDoVOTests {

    @Test("Deve criar com valores válidos")
    func validCreation() throws {
        let vo = try NomeDoVO(/* params válidos */)
        #expect(vo.property == expectedValue)
    }

    @Test("Deve rejeitar quando [condição]")
    func rejectCondition() {
        #expect(throws: NomeDoVOError.specificError) {
            try NomeDoVO(/* params inválidos */)
        }
    }
}
```

### Test Doubles disponíveis

| Double | Tipo | Uso |
|--------|------|-----|
| `InMemoryPatientRepository` | Repository | Simula persistência em memória |
| `InMemoryEventBus` | EventBus | Captura eventos emitidos para verificação |
| `InMemoryLookupValidator` | LookupValidator | Valida flags de lookup tables |
| `PatientFixture` | Factory | Cria pacientes com dados válidos para testes |

---

## 10. CHECKLIST MASTER — COBERTURA TOTAL

### Camada 1: Value Objects (~150 testes)

| VO | Happy | Norm. | Vazio | Formato | Negócio | Igualdade | Min |
|----|:-----:|:-----:|:-----:|:-------:|:-------:|:---------:|:---:|
| CPF | ◻ | ◻ | ◻ | ◻ | ◻ | ◻ | 8 |
| NIS | ◻ | ◻ | ◻ | ◻ | — | ◻ | 4 |
| CEP | ◻ | ◻ | ◻ | ◻ | ◻ | ◻ | 7 |
| PersonId | ◻ | ◻ | ◻ | ◻ | — | ◻ | 5 |
| ProfessionalId | ◻ | ◻ | ◻ | ◻ | — | ◻ | 5 |
| LookupId | ◻ | ◻ | ◻ | ◻ | — | ◻ | 5 |
| PatientId | ◻ | ◻ | ◻ | ◻ | — | ◻ | 5 |
| AppointmentId | ◻ | ◻ | ◻ | ◻ | — | ◻ | 5 |
| ReferralId | ◻ | ◻ | ◻ | ◻ | — | ◻ | 5 |
| ViolationReportId | ◻ | ◻ | ◻ | ◻ | — | ◻ | 5 |
| RGDocument | ◻ | ◻ | ◻ | ◻ | ◻ | — | 8 |
| Address | ◻ | ◻ | ◻ | — | ◻ | — | 7 |
| TimeStamp | ◻ | — | ◻ | ◻ | ◻ | ◻ | 5 |
| PersonalData | ◻ | ◻ | ◻×5 | — | ◻ | — | 9 |
| CivilDocuments | ◻ | — | — | — | ◻ | — | 3 |
| SocialIdentity | ◻ | — | — | — | ◻ | — | 4 |
| FamilyMember | ◻ | — | — | — | ◻ | ◻ | 3 |
| HousingCondition | ◻ | — | — | — | ◻ | — | 6 |
| SocioEconSituation | ◻ | — | ◻ | — | ◻ | — | 7 |
| SocialBenefit | ◻ | ◻ | ◻ | — | ◻ | — | 5 |
| SocialBenefitsColl | ◻ | — | — | — | ◻ | — | 3 |
| WorkIncomeVO | ◻ | — | — | — | ◻ | — | 3 |
| CommunitySuppNet | ◻ | — | — | — | ◻ | — | 4 |
| SocialHealthSum | ◻ | — | — | — | ◻ | — | 4 |
| HealthStatus | ◻ | — | — | — | — | — | 1 |
| EducationalStatus | ◻ | — | — | — | — | — | 1 |
| ICDCode | ◻ | ◻ | ◻ | — | ◻ | — | 5 |
| Diagnosis | ◻ | — | ◻ | — | ◻ | — | 3 |
| IngressInfo | ◻ | — | ◻ | — | — | — | 2 |
| SocialCareAppt | ◻ | — | — | — | ◻ | — | 5 |
| Referral | ◻ | — | ◻ | — | ◻×4 | — | 7 |
| ViolationReport | ◻ | — | ◻ | — | ◻ | — | 5 |
| PlacementRegistry | ◻ | — | — | — | ◻ | — | 3 |

### Camada 2: Agregados (~20 testes)
- Patient: 9 invariantes × 2 (happy + rejeição) = ~18
- Referral state machine: 5 transições = ~5

### Camada 3: Analytics (~15 testes)
- HousingAnalytics: 4
- FinancialAnalytics: 3
- FamilyAnalytics: 3
- EducationAnalytics: 5

### Camada 4: Use Cases (~70 testes)
- 17 use cases × ~4 cenários (happy, validação, not found, invariante) = ~68

### Camada 5: Cross-Validations (~15 testes)
- 4 cross-field × 2 (happy + rejeição) = 8
- 3 metadata-driven × 2 = 6

### Camada 6: DTOs/HTTP (~80 testes)
- 22 endpoints × ~3.5 cenários = ~77

### Camada 7: Eventos (~30 testes)
- 15 eventos × 2 (metadata + payload) = ~30

### **TOTAL ESTIMADO: ~380 testes**

---

## 11. EXEMPLOS DE CÓDIGO PARA CADA CAMADA

> Os exemplos completos de código para VOs estão em `shared/validation-rules/TESTING_GUIDE.md`. Aqui incluímos exemplos para as camadas adicionais.

### Camada 2 — Agregado Patient

```swift
@Suite("Patient - Invariantes")
struct PatientInvariantTests {

    @Test("Deve criar paciente com dados válidos")
    func validPatient() async throws {
        let repo = InMemoryPatientRepository()
        let bus = InMemoryEventBus()
        let handler = RegisterPatientUseCase(repository: repo, eventBus: bus)

        let command = RegisterPatientCommand(
            personId: PersonId(),
            initialDiagnoses: [try Diagnosis(id: try ICDCode("B201"), date: .now, description: "Doença genética rara diagnosticada", now: .now)],
            prRelationshipId: try LookupId(UUID().uuidString),
            personalData: /* PersonalData válido */,
            civilDocuments: try CivilDocuments(cpf: try CPF("52998224725")),
            address: /* Address válido */,
            socialIdentity: /* SocialIdentity válido */,
            actorId: UUID().uuidString
        )
        let result = try await handler.handle(command)
        #expect(!result.id.value.isEmpty)
        #expect(bus.publishedEvents.count == 1)
    }

    @Test("Deve rejeitar paciente sem diagnósticos")
    func rejectNoDiagnoses() async {
        let handler = RegisterPatientUseCase(/* ... */)
        let command = RegisterPatientCommand(
            /* ... initialDiagnoses: [] ... */
        )
        await #expect(throws: AppError.self) {
            try await handler.handle(command)
        }
    }

    @Test("Deve rejeitar adição de membro com personId duplicado")
    func rejectDuplicateMember() async {
        // 1. Criar paciente
        // 2. Adicionar membro com personId X
        // 3. Tentar adicionar outro membro com mesmo personId X → erro PAT-003
    }

    @Test("Deve rejeitar referral com alvo fora do boundary")
    func rejectReferralOutsideBoundary() async {
        // 1. Criar paciente com membro A
        // 2. Criar referral com referredPersonId = personId aleatório → erro PAT-006
    }
}
```

### Camada 3 — Analytics

```swift
@Suite("HousingAnalyticsService - Cálculos")
struct HousingAnalyticsTests {

    @Test("density(6 membros, 2 quartos) = 3.0")
    func normalDensity() {
        let d = HousingAnalyticsService.density(forMembers: 6, inBedrooms: 2)
        #expect(d == 3.0)
    }

    @Test("density(0 membros, 0 quartos) = 1.0 (safe division)")
    func safeDivision() {
        let d = HousingAnalyticsService.density(forMembers: 0, inBedrooms: 0)
        #expect(d == 1.0)
    }

    @Test("isOvercrowded = true quando density > 3.0")
    func overcrowded() {
        #expect(HousingAnalyticsService.isOvercrowded(members: 7, bedrooms: 2) == true)
    }

    @Test("isOvercrowded = false quando density == 3.0 (limite exato)")
    func notOvercrowdedAtLimit() {
        #expect(HousingAnalyticsService.isOvercrowded(members: 6, bedrooms: 2) == false)
    }
}

@Suite("FinancialAnalyticsService - Cálculos")
struct FinancialAnalyticsTests {

    @Test("Deve calcular todos os indicadores financeiros")
    func allIndicators() {
        let r = FinancialAnalyticsService.calculate(
            workIncomes: [1000.0, 500.0],
            socialBenefitAmounts: [600.0],
            memberCount: 4
        )
        #expect(r.totalWorkIncome == 1500.0)
        #expect(r.perCapitaWorkIncome == 375.0)
        #expect(r.totalGlobalIncome == 2100.0)
        #expect(r.perCapitaGlobalIncome == 525.0)
    }

    @Test("Deve usar memberCount mínimo 1 para evitar divisão por zero")
    func safeMinimum() {
        let r = FinancialAnalyticsService.calculate(
            workIncomes: [1000.0], socialBenefitAmounts: [], memberCount: 0
        )
        #expect(r.perCapitaWorkIncome == 1000.0)
    }
}
```

### Camada 4 — Use Case

```swift
@Suite("AddFamilyMember - Use Case")
struct AddFamilyMemberUseCaseTests {

    @Test("Deve adicionar membro e emitir evento")
    func happyPath() async throws {
        let (repo, bus, handler) = makeHandler()
        let patient = try await createPatient(repo: repo)

        let command = AddFamilyMemberCommand(
            patientId: patient.id,
            memberPersonId: PersonId(),
            relationship: try LookupId(UUID().uuidString),
            isResiding: true,
            isCaregiver: false,
            hasDisability: false,
            requiredDocuments: [],
            birthDate: try TimeStamp(iso: "2010-05-15T00:00:00.000Z"),
            prRelationshipId: nil,
            actorId: UUID().uuidString
        )
        try await handler.handle(command)

        let updated = try await repo.find(by: patient.id)
        #expect(updated?.familyMembers.count == 1)
        #expect(bus.publishedEvents.last is FamilyMemberAddedEvent)
    }

    @Test("Deve rejeitar quando paciente não existe")
    func patientNotFound() async {
        let (_, _, handler) = makeHandler()
        let command = AddFamilyMemberCommand(
            patientId: PatientId(), /* ... */
        )
        await #expect(throws: AppError.self) {
            try await handler.handle(command)
        }
    }
}
```

### Camada 5 — Cross-Validation

```swift
@Suite("CrossValidator - Validações inter-campo")
struct CrossValidatorTests {

    @Test("Deve rejeitar gestante do sexo masculino")
    func pregnancyRequiresFemale() {
        let maleMember = FamilyMember(/* sex: .masculino */)
        let healthStatus = HealthStatus(gestatingMembers: [maleMember.personId], /* ... */)

        #expect(throws: AppError.self) {
            try CrossValidator.validatePregnancySex(
                gestatingMembers: healthStatus.gestatingMembers,
                familyMembers: [maleMember]
            )
        }
    }

    @Test("Deve aceitar gestante do sexo feminino")
    func pregnancyFemaleOk() throws {
        let femaleMember = FamilyMember(/* sex: .feminino */)
        try CrossValidator.validatePregnancySex(
            gestatingMembers: [femaleMember.personId],
            familyMembers: [femaleMember]
        )
    }

    @Test("Deve rejeitar thirdPartyGuardReport sem menor de 18")
    func guardRequiresMinor() {
        let adultMember = FamilyMember(/* birthDate: 30 anos atrás */)
        #expect(throws: AppError.self) {
            try CrossValidator.validateThirdPartyGuard(
                thirdPartyGuardReport: true,
                familyMembers: [adultMember],
                referenceDate: .now
            )
        }
    }

    @Test("Deve rejeitar adolescentInInternment sem membro 12-17")
    func internmentRequires12to17() {
        let childMember = FamilyMember(/* birthDate: 5 anos atrás */)
        #expect(throws: AppError.self) {
            try CrossValidator.validateAdolescentInternment(
                adolescentInInternment: true,
                familyMembers: [childMember],
                referenceDate: .now
            )
        }
    }
}
```

### Camada 6 — DTOs e HTTP

```swift
@Suite("RegisterPatientRequest - DTO")
struct RegisterPatientRequestTests {

    @Test("Deve desserializar JSON válido completo")
    func validJSON() throws {
        let json = """
        {
            "personId": "550e8400-e29b-41d4-a716-446655440000",
            "initialDiagnoses": [{
                "icdCode": "B20.1",
                "date": "2025-01-01",
                "description": "Diagnóstico de doença genética rara confirmado por exame"
            }],
            "prRelationshipId": "550e8400-e29b-41d4-a716-446655440001",
            "personalData": { /* ... */ },
            "civilDocuments": { "cpf": "529.982.247-25" },
            "address": { "state": "SP", "city": "São Paulo", "isShelter": false, "residenceLocation": "URBANO" },
            "socialIdentity": { "typeId": "550e8400-e29b-41d4-a716-446655440002" }
        }
        """.data(using: .utf8)!

        let dto = try JSONDecoder().decode(RegisterPatientRequestDTO.self, from: json)
        #expect(dto.personId == "550e8400-e29b-41d4-a716-446655440000")
        #expect(dto.initialDiagnoses.count == 1)
    }

    @Test("Deve rejeitar JSON sem campo obrigatório personId")
    func missingRequired() {
        let json = """
        { "initialDiagnoses": [{ "icdCode": "B20.1", "date": "2025-01-01", "description": "Teste de diagnóstico" }] }
        """.data(using: .utf8)!

        #expect(throws: DecodingError.self) {
            try JSONDecoder().decode(RegisterPatientRequestDTO.self, from: json)
        }
    }
}
```

### Camada 7 — Eventos

```swift
@Suite("Eventos - Payload e Metadata")
struct EventPayloadTests {

    @Test("PatientCreated deve conter metadata e dados corretos")
    func patientCreatedEvent() async throws {
        let bus = InMemoryEventBus()
        // ... executar RegisterPatientUseCase ...

        let event = bus.publishedEvents.first as! PatientCreatedEvent
        #expect(event.metadata.eventId != nil)
        #expect(event.metadata.occurredAt != nil)
        #expect(event.metadata.schemaVersion == "1.0.0")
        #expect(event.data.patientId != nil)
        #expect(event.data.personId != nil)
    }

    @Test("AssessmentUpdated deve conter before e after snapshots")
    func assessmentUpdatedEvent() async throws {
        let bus = InMemoryEventBus()
        // ... executar UpdateHousingConditionUseCase ...

        let event = bus.publishedEvents.last as! AssessmentUpdatedEvent
        #expect(event.data.patientId != nil)
        // before pode ser nil na primeira atualização
        #expect(event.data.after != nil)
    }
}
```

---

## 12. COMO USAR ESTE GUIA (Passo a Passo para IA)

### Passo 1 — Ler os contratos
1. Leia este guia (`AI_TESTING_GUIDE.md`) inteiro
2. Para detalhes de um schema específico, leia o arquivo YAML em `services/social-care/model/schemas/`
3. Para regras de validação detalhadas, leia o YAML correspondente em `shared/validation-rules/`

### Passo 2 — Ler o código existente
1. Leia a implementação do VO/use case em `social-care/Sources/social-care-s/`
2. Leia os testes existentes em `social-care/Tests/`
3. Identifique quais regras já estão testadas e quais faltam

### Passo 3 — Implementar os testes
1. Use `swift-testing` (NUNCA XCTest)
2. Siga as convenções da seção 9
3. Para cada VO/use case, implemente todas as categorias de teste da seção 8
4. Use os test doubles existentes (InMemoryPatientRepository, InMemoryEventBus, etc.)

### Passo 4 — Verificar cobertura
1. Use o checklist da seção 10 para rastrear progresso
2. Execute `make coverage` para verificar cobertura
3. O CI exige 95% de cobertura

### Passo 5 — Validar contra o contrato
Para cada teste implementado, confirme que:
- O campo/regra existe no schema YAML correspondente
- O tipo de dado está correto (string, number, boolean, enum)
- Os campos required/optional estão de acordo com o schema
- Os valores de enum são exatamente os definidos no YAML
- As mensagens de erro usam o código padronizado (ex: `PAT-001`)
- Os eventos emitidos correspondem ao canal e payload do AsyncAPI

---

## 13. REFERÊNCIA RÁPIDA DE ARQUIVOS

| O que procurar | Onde ler |
|----------------|---------|
| Endpoints HTTP | `services/social-care/openapi/openapi.yaml` |
| Eventos assíncronos | `services/social-care/asyncapi/asyncapi.yaml` |
| Schema de um DTO específico | `services/social-care/model/schemas/<Nome>.yaml` |
| Regras de validação de kernel VOs | `shared/validation-rules/kernel.yaml` |
| Regras de validação de registry | `shared/validation-rules/registry.yaml` |
| Regras de validação de assessment | `shared/validation-rules/assessment.yaml` |
| Regras de validação de care | `shared/validation-rules/care.yaml` |
| Regras de validação de protection | `shared/validation-rules/protection.yaml` |
| Indicadores calculados | `shared/validation-rules/analytics.yaml` |
| Regras inter-campo | `shared/validation-rules/cross-validations.yaml` |
| Algoritmos custom (CPF checksum, etc.) | `shared/validation-rules/_custom-validators.yaml` |
| Schemas de erro compartilhados | `shared/schemas/Error.yaml` |
| Paginação | `shared/schemas/Pagination.yaml` |
| Exemplos de código para VOs | `shared/validation-rules/TESTING_GUIDE.md` |
