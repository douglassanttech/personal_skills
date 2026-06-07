# Plano de Otimização de Performance — Fortius (Banco de Dados)

**Data:** 2026-06-07
**Contexto:** Instância Cloud SQL `fortius-prod` em `db-f1-micro` (0.6 GB RAM) com memória a 100% e alertas de underprovisioned resource. Upgradada para `db-g1-small` (1.7 GB). Análise feita via `pg_stat_statements`.

---

## Diagnóstico

### Infra
- `fortius-prod`: PostgreSQL 16, `db-g1-small`, 1.7 GB RAM, 20 GB disco
- `fortius-qa`: PostgreSQL 16, `db-g1-small`, 20 GB disco
- `pg_stat_statements` habilitado em 2026-06-07

### Principais problemas encontrados

| Tabela | Problema | Métrica |
|--------|----------|---------|
| `medical_records` (list) | Sem paginação, traz tudo com JOINs + campos grandes | spill to disk: 1M temp_blks, avg 262ms, max 1996ms |
| `medical_records` (list) | 5 queries em cascata antes da query principal | +5 roundtrips por request |
| `medical_record_attachments` | Sem índice em `medicalRecordId` | 132k seq scans, 99% seq scan |
| `medical_records` | Sem índice em `(patientId, clinicId)` e `(clinicId, recordDate)` | COUNTs e listagens sem índice |
| `whatsapp_messages` | 67M blocos lidos, 11.8k chamadas | Possível polling excessivo |
| `domain_entity_events` | 100% seq scan, sem índice | 816 scans, tabela de 15 MB |

---

## Ações por Prioridade

### ✅ 1. Upgrade de tier — CONCLUÍDO
- `db-f1-micro` → `db-g1-small` (0.6 GB → 1.7 GB RAM)
- Downtime: ~2 min
- Custo: ~$7/mês → ~$25/mês

### ✅ 2. Consolidar queries em cascata no list de prontuários — CONCLUÍDO
- **Arquivo:** `fortius_be/src/use_cases/medical-records/queries/list-medical-records.query-handler.ts`
- **Antes:** 5 roundtrips ao banco (findOne clinic, findOne patient, findOne patientClinic, find medicalRecords p/ clinicIds, find patientClinics)
- **Depois:** 1 query SQL com subqueries (`EXISTS` + `ARRAY + UNION`) + query principal
- **Risco:** Zero — comportamento idêntico

### ⏳ 3. Criar índices via migration — MIGRATION CRIADA, PENDENTE DE RODAR
- **Arquivo:** `fortius_be/src/migrations/1769115000000-add-performance-indexes.ts`
- Índices:
  - `medical_records(patientId, clinicId)` — query principal de listagem
  - `medical_records(clinicId, recordDate)` — COUNTs por data
  - `medical_record_attachments(medicalRecordId)` — 132k seq scans
- Usar `CONCURRENTLY` para não travar tabela em prod
- **Risco:** Zero

### ⏳ 4. Paginação no list de prontuários — PENDENTE
- **BE:** adicionar `take/skip` no `medicalRecordRepository.find()`
- **FE:** scroll infinito ou botão "carregar mais" na lista do `PatientDetail.tsx`
- **Impacto:** elimina o spill to disk na query principal
- **Risco:** Médio — requer mudança coordenada BE + FE
- **Observação:** sem paginação, mesmo com índice, buscar 100+ prontuários por paciente pode continuar lento

### ⏳ 5. Remover `content`/`evaluationFields` da resposta da listagem — PENDENTE
- **BE:** usar `select` no find para excluir campos grandes da lista
- **FE:** `getSummary()` em `PatientDetail.tsx` (linha 1453) usa `evaluationFields` diretamente — precisa ser ajustado para usar apenas o campo `summary` que já vem pré-montado pelo backend (`buildSummary`)
- **Impacto:** reduz drasticamente o payload e a memória usada no sort
- **Risco:** Baixo — `getContentHtml()` já prioriza `summary`, só `getSummary()` precisa ser ajustado

### ⏳ 6. Investigar polling em `whatsapp_messages` — PENDENTE
- 11.874 chamadas lendo 67M blocos — índice `(clinicId, phoneNumber)` já existe
- Verificar se há `setInterval` ou polling no frontend/agente de atendimento consultando mensagens em loop
- Avaliar uso de WebSocket ou SSE para substituir polling

### ⏳ 7. Índice em `domain_entity_events` — PENDENTE
- 100% seq scan, 816 chamadas, tabela de 15 MB
- Investigar quais queries batem nessa tabela e quais colunas filtram
- Criar índice adequado via nova migration

---

## Como rodar a migration em prod

```bash
cd fortius_be
DB_HOST=34.58.103.77 DB_PORT=5432 DB_USER=postgres DB_PASSWORD=<senha> DB_NAME=fortius DB_SSL=true npm run migration:run
```

**Lembrar:** liberar IP local no Cloud SQL antes (`npm run db:allow-ip` com `IP=<seu_ip>`).

---

## Referências
- Handler otimizado: `fortius_be/src/use_cases/medical-records/queries/list-medical-records.query-handler.ts`
- Migration de índices: `fortius_be/src/migrations/1769115000000-add-performance-indexes.ts`
- Frontend afetado: `fortius_fe/src/pages/PatientDetail.tsx` (funções `getSummary` e `getContentHtml`)
