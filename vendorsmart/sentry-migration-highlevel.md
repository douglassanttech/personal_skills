 Análise dos Alertas Sentry → VendorSmart Workflow

  A planilha tem 52 alertas do sistema legado. As colunas "Data Entry", "Manager", "Payable", "Paid" já indicam em qual estágio do processo atual cada alerta aparece. Precisamos classificar cada um em:

  - Workflow Step — uma etapa real de aprovação no fluxo
  - Tag/Rule — uma condição que adiciona tag ou dispara regra dentro de um step existente
  - System Validation — checagem automática do sistema (não depende de humano)
  - Vendor/Bank Config — configuração no cadastro, não no workflow
  - Not Used / Future — itens que a Sentry não usa ou ficam para waves futuras

  ---
  WORKFLOW STEPS (etapas reais de aprovação)

  Estes alertas representam filas de aprovação distintas com pessoas/papéis diferentes. Cada um vira um step no workflow:

  ┌─────┬────────────────────┬──────────────────────┬────────────────────────────┬──────────────────────────────────────────────────────┐
  │  #  │        Step        │     Quem aprova      │ Threshold (invoice amount) │                      Observação                      │
  ├─────┼────────────────────┼──────────────────────┼────────────────────────────┼──────────────────────────────────────────────────────┤
  │ 1   │ Data Entry         │ AP Clerk             │ —                          │ Preenchimento de dados, approvers: 0                 │
  ├─────┼────────────────────┼──────────────────────┼────────────────────────────┼──────────────────────────────────────────────────────┤
  │ 2   │ Coding             │ AP Clerk             │ —                          │ GL distribution, approvers: 0                        │
  ├─────┼────────────────────┼──────────────────────┼────────────────────────────┼──────────────────────────────────────────────────────┤
  │ 3   │ AP Review          │ AP Supervisor        │ —                          │ Revisão geral AP (dismiss de maioria dos alertas AP) │
  ├─────┼────────────────────┼──────────────────────┼────────────────────────────┼──────────────────────────────────────────────────────┤
  │ 4   │ Manager Approval   │ Property Manager     │ —                          │ Aprovação do gestor da association                   │
  ├─────┼────────────────────┼──────────────────────┼────────────────────────────┼──────────────────────────────────────────────────────┤
  │ 5   │ VP Approval        │ Vice President       │ $10,000 – $24,999          │ Threshold-based, auto-skip abaixo                    │
  ├─────┼────────────────────┼──────────────────────┼────────────────────────────┼──────────────────────────────────────────────────────┤
  │ 6   │ SVP Approval       │ Sr. Vice President   │ $25,000 – $99,999          │ Threshold-based                                      │
  ├─────┼────────────────────┼──────────────────────┼────────────────────────────┼──────────────────────────────────────────────────────┤
  │ 7   │ EVP Approval       │ Exec. Vice President │ $100,000 – $249,999        │ Threshold-based                                      │
  ├─────┼────────────────────┼──────────────────────┼────────────────────────────┼──────────────────────────────────────────────────────┤
  │ 8   │ President Approval │ President            │ $250,000+                  │ Threshold-based                                      │
  └─────┴────────────────────┴──────────────────────┴────────────────────────────┴──────────────────────────────────────────────────────┘

  Por que são steps:
  - Cada um tem uma fila de aprovação separada ("dismissed via X Review queue approval")
  - Representam níveis hierárquicos com thresholds de valor crescentes
  - Têm pessoas/papéis distintos responsáveis

  Opcionais (Wave 2+):

  ┌─────┬────────────────────┬──────────────────────────────────┬──────────┐
  │  #  │        Step        │              Quando              │   Wave   │
  ├─────┼────────────────────┼──────────────────────────────────┼──────────┤
  │ 9   │ Banking Review     │ Problemas de conta bancária      │ Wave 1-2 │
  ├─────┼────────────────────┼──────────────────────────────────┼──────────┤
  │ 10  │ Cash Mgmt Approval │ Cash Poor / Manager Hold         │ Wave 1   │
  ├─────┼────────────────────┼──────────────────────────────────┼──────────┤
  │ 11  │ Audit Review       │ Seleção aleatória para auditoria │ Wave 2   │
  └─────┴────────────────────┴──────────────────────────────────┴──────────┘

  ---
  TAGS + RULES (condições dentro de steps, NÃO são steps)

  Estes alertas viram tags no invoice + regras condicionais nos step configs:

  ┌─────────────────────────────┬─────────────────────────┬───────────────────────────────────────────────────────────────────────────────┬──────┐
  │           Alerta            │          Tipo           │                               Como implementar                                │ Wave │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Duplicate Invoice Number    │ System validation + tag │ Tag Duplicate → envia para AP Review                                          │ W1   │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Owner Reimbursement         │ Tag + rule              │ Tag Owner Reimbursement → step extra de aprovação. Consumir vendor type do C3 │ W1   │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Purchase Only               │ Tag                     │ Tag Purchase Only, sem step especial                                          │ W1   │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Borrow From Reserves        │ Tag (manager adds)      │ Tag Borrow From Reserves → fila de aprovação                                  │ W1   │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Credit Account (non-ACH)    │ Tag + rule              │ Tag Credit Account → AP Review. Credit memo feature                           │ W1   │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Credit Account/ACH          │ Tag                     │ $0 voucher + notas. Tag Credit ACH                                            │ W1   │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ ACH/Utility Back Dated      │ Tag + rule              │ Tag Back Dated → precisa inserir payment date. Prepaid logic                  │ W1   │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Request for Cashier's Check │ Tag                     │ Tag Cashier's Check → Manager + Payable review                                │ W1   │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Priority Invoice            │ Tag                     │ Tag Priority → VP approval                                                    │ —    │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ One Time Pay                │ Tag                     │ Tag One Time Pay → SVP approval (vendor sem insurance)                        │ —    │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Rejected                    │ Status                  │ Invoice status = REJECTED, volta para step anterior                           │ —    │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Data Entry Level 1          │ Rule                    │ Rule: se keyed por DE Level 1 → Review queue adicional                        │ —    │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Employee Reimbursement      │ Tag + rule              │ Tag Employee Reimbursement → VP approval                                      │ —    │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Partial Pay                 │ Feature                 │ Funcionalidade de pagamento parcial, não é step                               │ W2   │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Manager Hold                │ Status/Tag              │ Tag On Hold → bloqueia progressão                                             │ —    │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ IRS Invoice Hold            │ Vendor flag + tag       │ Flag no vendor → tag IRS Hold → AP Review                                     │ W2   │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Restricted Vendor           │ Vendor flag + tag       │ Flag no vendor C3 → tag Restricted Vendor                                     │ W2   │
  ├─────────────────────────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────────────────┼──────┤
  │ Vendor on Executive Hold    │ Vendor flag + tag       │ Flag no vendor → tag → SVP/President review                                   │ W2   │
  └─────────────────────────────┴─────────────────────────┴───────────────────────────────────────────────────────────────────────────────┴──────┘

  Por que NÃO são steps:
  - São condições que podem ocorrer em qualquer step
  - Resolvidos com tags que disparam regras nos step configs existentes
  - Não têm fila de aprovação própria — são revisados dentro de steps já existentes

  ---
  SYSTEM VALIDATIONS / VENDOR CONFIG (não é workflow)

  ┌─────────────────────────────────────┬───────────────────────────────────┬───────────────────────────────────────────────────────┐
  │               Alerta                │         Como implementar          │                  Por quê não é step                   │
  ├─────────────────────────────────────┼───────────────────────────────────┼───────────────────────────────────────────────────────┤
  │ W9 Missing                          │ Validação no cadastro do vendor   │ Resolvido uploadando documento, não aprovando invoice │
  ├─────────────────────────────────────┼───────────────────────────────────┼───────────────────────────────────────────────────────┤
  │ Liability Insurance Missing/Expired │ Validação no cadastro do vendor   │ Idem                                                  │
  ├─────────────────────────────────────┼───────────────────────────────────┼───────────────────────────────────────────────────────┤
  │ Workers Comp Missing/Expired        │ Validação no cadastro do vendor   │ Idem                                                  │
  ├─────────────────────────────────────┼───────────────────────────────────┼───────────────────────────────────────────────────────┤
  │ Account Closing                     │ Status da bank account (inactive) │ Não exibir contas inativas no dropdown                │
  ├─────────────────────────────────────┼───────────────────────────────────┼───────────────────────────────────────────────────────┤
  │ Fraud Warning                       │ Flag na bank account              │ Alerta visual, não step de aprovação                  │
  ├─────────────────────────────────────┼───────────────────────────────────┼───────────────────────────────────────────────────────┤
  │ Closed Bank COA                     │ Status da bank account            │ Validação sistêmica                                   │
  ├─────────────────────────────────────┼───────────────────────────────────┼───────────────────────────────────────────────────────┤
  │ Canceled Property                   │ Status da association             │ Bloquear invoices de properties canceladas            │
  ├─────────────────────────────────────┼───────────────────────────────────┼───────────────────────────────────────────────────────┤
  │ Transaction Limit Met               │ Validação de limite bancário      │ Sistêmico, não humano                                 │
  ├─────────────────────────────────────┼───────────────────────────────────┼───────────────────────────────────────────────────────┤
  │ Cash Poor                           │ Validação de saldo mínimo         │ Bloqueia payment, não é step                          │
  └─────────────────────────────────────┴───────────────────────────────────┴───────────────────────────────────────────────────────┘

  ---
  NOT USED / DESCARTADOS

  ┌───────────────────────────────┬────────────────────┐
  │            Alerta             │       Motivo       │
  ├───────────────────────────────┼────────────────────┤
  │ Credit Voucher                │ Marcado "Not Used" │
  ├───────────────────────────────┼────────────────────┤
  │ Bank/Branch/Account Creation  │ Not Used           │
  ├───────────────────────────────┼────────────────────┤
  │ BOD Only Account              │ Not Used           │
  ├───────────────────────────────┼────────────────────┤
  │ Cash Management Creation      │ Not Used           │
  ├───────────────────────────────┼────────────────────┤
  │ Bank COA Does not cash checks │ Not Used           │
  ├───────────────────────────────┼────────────────────┤
  │ COA Creation Required         │ Not Used           │
  ├───────────────────────────────┼────────────────────┤
  │ Selection Property            │ Not Used           │
  ├───────────────────────────────┼────────────────────┤
  │ Missing PAxtX File            │ DL4 legacy, Wave 3 │
  └───────────────────────────────┴────────────────────┘

  ---
  Proposta de Workflow — Sentry (Wave 1)

  Step 1: Data Entry          (approvers: 0)  — EDIT: Invoice Data, Distribution
  Step 2: Coding              (approvers: 0)  — EDIT: Distribution, Bank Info
  Step 3: AP Review           (approvers: 1)  — Dismisses AP alerts/tags
  Step 4: Manager Approval    (approvers: 1)  — Property manager review
  Step 5: VP Approval         (approvers: 1, threshold: $10,000)
  Step 6: SVP Approval        (approvers: 1, threshold: $25,000)
  Step 7: EVP Approval        (approvers: 1, threshold: $100,000)
  Step 8: President Approval  (approvers: 1, threshold: $250,000)

  Rules nos steps:
  - Step 3 (AP Review): rules para tags Duplicate, Credit Account, Back Dated, Cashier's Check
  - Step 4 (Manager): rules para tags Borrow From Reserves, Owner Reimbursement
  - Step 5-8: auto-skip via threshold — invoice de $5k pula direto do Manager para Payable

  Rejection config: rejection em qualquer step → volta para Step 2 (Coding) com reasonRequired: true
