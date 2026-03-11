# Validation Rules — Social Care

Fonte única de verdade para regras de validação do domínio `social-care`.
Qualquer BFF, frontend ou serviço consumidor **deve** implementar estas mesmas regras.

## Estrutura

```
validation-rules/
  _custom-validators.yaml   # Algoritmos que cada plataforma implementa nativamente
  kernel.yaml               # VOs cross-cutting (CPF, NIS, CEP, etc.)
  registry.yaml             # Bounded context Registry (Patient, FamilyMember, etc.)
  assessment.yaml           # Bounded context Assessment (Housing, Health, etc.)
  care.yaml                 # Bounded context Care (Appointment, Diagnosis, etc.)
  protection.yaml           # Bounded context Protection (Referral, Violation, etc.)
  cross-validations.yaml    # Regras cross-field e metadata-driven
  analytics.yaml            # Indicadores calculados (housing density, financial, etc.)
```

## Convenções dos YAMLs

- `required: true` → campo obrigatório
- `normalize` → transformação aplicada ANTES da validação
- `rules[]` → lista ordenada de validações (executar na ordem)
- `error_code` → código estável para suporte (ex: `CPF-001`)
- `http_status` → status HTTP sugerido (default 422)
- `cross_validations[]` → regras que dependem de mais de um campo
- `enums[]` → valores válidos para campos enumerados
