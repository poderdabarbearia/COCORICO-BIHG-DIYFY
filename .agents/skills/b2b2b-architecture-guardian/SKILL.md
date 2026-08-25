---
name: b2b2b-architecture-guardian
description: >
  Arquitetura e proteção para transformar SaaS multiusuário/multiempresa em B2B2B,
  adicionando uma camada de Provedores/Parceiros acima dos tenants existentes sem
  quebrar, duplicar ou reescrever os módulos operacionais já funcionando.
version: 1.0
canonical_name: poder-b2b2b-architecture-guardian
---

# B2B2B Architecture Guardian

## 1. Missão
Esta Skill transforma com segurança um SaaS multiusuário/multiempresa em B2B2B: `PLATFORM/MASTER → PROVIDER/PARTNER/RESELLER/FRANCHISEE → TENANT/COMPANY/UNIT/BUSINESS → USERS/TEAM → OPERAÇÃO`. Funciona em qualquer nicho. Provider é uma camada acima do tenant e NÃO substitui o tenant. O objetivo é adicionar distribuição, revenda, franquia ou gestão em rede sem prejudicar o sistema existente.

## 2. Regra máxima
> NUNCA transformar um sistema multiempresa em B2B2B reescrevendo o núcleo operacional se o isolamento atual por tenant já funciona.
Antes de alterar: identificar tenant real, acesso atual, função central de autorização, tabelas operacionais, billing, Auth, RLS, RPCs, Edge Functions, workers, WhatsApp e Storage. Preservar tudo que funciona e adicionar Provider ACIMA. Estratégia inicial sempre ADITIVA.

## 3. Modelo canônico
`Platform → Providers → Tenants → Tenant Users → Dados operacionais`, mantendo também `Platform → tenants diretos`. Restaurante: Franqueado→Restaurante; Academia: Rede→Academia; Clínica: Licenciado→Clínica; Barbearia: Provedor→Barbearia. O conceito é o mesmo.

## 4. Entidades mínimas
Preferir conceitos `provider`, `provider_user`, `tenant.provider_id`, ou equivalentes existentes como `reseller`, `reseller_user`, `company.reseller_id`. Nunca renomear tabelas saudáveis só por nomenclatura. Provider representa parceiro/franqueado/licenciado/distribuidor/revendedor. Provider User é membership independente da equipe do tenant; papéis típicos owner/admin/support/finance. Não reutilizar `tenant_user`/`company_user`. Tenant continua sendo a unidade operacional; normalmente `provider_id NULLABLE`, onde NULL = cliente direto.

## 5. Compatibilidade
`tenant.provider_id = NULL` deve representar exatamente o comportamento antigo. Clientes atuais, onboarding, billing direto, RLS, dashboards, agenda, vendas, estoque e módulos ativos continuam funcionando. Migration B2B2B não pode obrigar tenants antigos a ganhar Provider.

## 6. Não propagar Provider
Se tabelas operacionais já usam `tenant_id` e tenant tem `provider_id`, resolver `registro → tenant_id → tenant.provider_id`. NÃO adicionar `provider_id` em appointment, sale, customer, product, service, financial_entry etc. sem motivo forte de performance, ledger, auditoria ou histórico imutável.

## 7. Memberships
`provider_user != tenant_user`. Provider Admin não vira tenant admin automaticamente. Acesso operacional exige fluxo separado e explícito.

## 8. Escopo no servidor
Nunca confiar em `request.provider_id`, `request.reseller_id`, `request.tenant_id` ou `request.company_id` para autorização. Resolver `auth.uid() → membership → provider atual → tenant pertence ao provider? → role permite?`. IDs do frontend identificam alvo, nunca autoridade.

## 9. Multi-provider
Escolher explicitamente: usuário em um único Provider com constraint; OU múltiplos Providers com `active_provider_id` explícito e validado server-side. PROIBIDO escolher silenciosamente com `ORDER BY created_at LIMIT 1` quando múltiplos vínculos forem possíveis.

## 10. Fronteira de autorização
Separar `has_tenant_access()`, `has_provider_access()`, `provider_owns_tenant()`, `is_provider_admin()`, `my_provider_context()`. NÃO adicionar automaticamente `provider_owns_tenant()` dentro de `has_tenant_access()`, pois pode abrir clientes, vendas, financeiro, agenda e demais tabelas ao Provider.

## 11. Matriz mínima
Master gerencia Providers, tenants diretos, hierarquia, auditoria, status e regras comerciais. Provider vê cadastro/equipe/carteira, cria tenants quando autorizado e administra escopo comercial permitido. Por padrão não lê PII, não movimenta caixa, não acessa financeiro, não altera agenda, não exclui dados e não vira tenant admin silenciosamente. Tenant opera como antes.

## 12. Suporte/impersonação
Suporte entre níveis é explícito, temporário, auditado, revogável, limitado e revalidado. Preferir `support_session(provider_id, tenant_id, actor_user_id, actor_role, reason, status, started_at, expires_at, ended_at)`. Nunca criar tenant_user temporário para suporte nem liberar Provider permanentemente em has_tenant_access. Preferir Support Session → Server Functions seguras → dados mínimos/redigidos. Escritas futuras usam RPCs/server functions dedicadas e auditadas.

## 13. Dados sensíveis
Por padrão não expor PII desnecessária, telefone/e-mail de cliente, documentos, histórico individual, pagamentos, itens de venda, financeiro detalhado, credenciais ou secrets.

## 14. Billing B2B2B
Não assumir `usuário = payer = seller = beneficiary`. Separar `billing_subject`, `billing_channel`, `payment_provider`, `seller`, `payer`, `beneficiary`. Subjects: tenant/provider. Channels: direct/provider/marketplace. Regras específicas pertencem ao billing-payment-guardian e adapters.

## 15. Revenue share
Quando houver split, modelar ledger explícito: gross_amount, payment_provider_fee, platform_share, provider_share, provider_net, currency, payment_id, split_verified, review_required. Nunca calcular repasse só no frontend nem liberar plano pelo redirect; autoridade é webhook/API oficial + operação idempotente.

## 16. Billing Provider != Billing Tenant
Manter domínios separados, como `provider_subscription` e `tenant_subscription`. Suspensão do Provider e inadimplência do Tenant são eventos diferentes; não reutilizar um único status para ambos.

## 17. Cascatas
Definir matriz explícita para suspensão/cancelamento/limites do Provider e efeitos nos tenants. Nunca usar UPDATE em massa sem necessidade; preferir estado derivado quando reversão deve ser imediata.

## 18. Direct vs Partner
`provider_id = NULL` = tenant direto; `provider_id = <id>` = tenant da carteira. Fluxos diretos antigos continuam. Provider não sequestra tenant direto por parâmetro do frontend. Vincular/desvincular é operação privilegiada e auditada.

## 19. Criação de tenant
Self-service direto: `provider_id = NULL` obrigatório. Provider cria tenant: provider_id vem do contexto autenticado no servidor. Master pode selecionar Provider com auditoria. Nunca usar provider_id do formulário como autoridade.

## 20. Ownership
Separar `created_by` de `provider_id`. O usuário Provider que cria empresa não vira owner operacional por created_by. Provisionamento segue contrato real de ownership sem acesso indireto.

## 21. White-label
Preferir herança `Tenant override → Provider config → Platform default` para marca, logo, cores, favicon, suporte, templates, domínio, knowledge, WhatsApp e textos. Não duplicar config do Provider em todos tenants.

## 22. Serviços compartilhados
Todo recurso global deve ter ownership explícito, preferindo `scope_type` + `scope_id` com platform/provider/tenant. Aplicável a WhatsApp, automações, filas, campanhas, knowledge, avisos, templates, integrações, relatórios, branding e notificações. Nunca misturar Provider A com B.

## 23. WhatsApp e integrações
Instâncias/configurações devem ter scope explícito. Workers e webhooks sempre reconstroem o escopo server-side.

## 24. Auditoria
Ações privilegiadas registram actor_user_id, actor_role, provider_id, tenant_id, ação, recurso, origem, timestamp e before/after seguro quando aplicável. Nunca gravar secrets ou PII desnecessária. Ações críticas exigem auditoria transacional quando a trilha for obrigatória.

## 25. Migração B2B → B2B2B
Fase 0 Auditoria. Fase 1 fundação aditiva (`provider`, `provider_user`, `tenant.provider_id nullable` + funções básicas + RLS mínima), com sistema antigo funcionando igual. Fase 2 Master. Fase 3 Provider Panel sem tocar telas operacionais. Fase 4 segurança. Fase 5 suporte. Fase 6 billing Provider. Fase 7 white-label/marketplace.

## 26. Preservação funcional
Inventariar auth, dashboard, agenda, clientes, equipe, estoque, vendas, checkout, financeiro, relatórios, assinaturas, notificações, WhatsApp e configurações. Após cada fase, regressão. Painel Provider funcionar não basta; tenant antigo deve continuar funcional.

## 27. Proibição de reimplementação
Se já existe agenda, vendas, billing, WhatsApp, estoque, Auth, clientes ou relatórios, NÃO criar segundo motor para Provider. Preferir `reusar → adaptar → proteger`, nunca `duplicar → migrar → substituir`.

## 28. Anti-patterns proibidos
Nunca: confiar em provider_id do frontend; transformar Provider em tenant admin; abrir função global de acesso sem análise; espalhar provider_id em tabelas; duplicar motores/Auth/tenant; copiar migrations cegamente; converter tenants de uma vez sem rollback; confundir gateway com Provider; misturar billings; usar redirect como prova de pagamento; escolher primeiro Provider ambiguamente; criar membership temporária para suporte; abrir RLS ampla; remover tenants diretos; quebrar módulos existentes para encaixar hierarquia.

## 29. Matriz de testes
Provider A→A permitido; A→B bloqueado; A→Tenant A conforme matriz; A→Tenant B bloqueado. Tenant A→A permitido; Tenant A→B bloqueado; Tenant→Provider bloqueado. Tenant direto continua; Provider→tenant direto bloqueado. Master conforme role/auditoria. Testar SELECT/INSERT/UPDATE/DELETE, RPC, server functions, Edge, Storage, workers, webhooks, WhatsApp, billing, exports, dashboards e logs.

## 30. Regression Gate
Validar Auth, tenant, onboarding direto, CRUD operacional, relatórios, agenda, vendas/checkout, estoque, billing direto, webhooks idempotentes, WhatsApp com escopo, mobile, isolamento Provider/Tenant, tenants diretos, ausência de duplicação, migrations compatíveis e build/typecheck/testes.

## 31. Uso com outras Skills
Sistema existente: `project-auditor → b2b2b-architecture-guardian → multi-tenant-guardian → supabase-guardian → safe-code-change → release-validator → deploy-forensics`.
Novo SaaS: `new-system-architect → b2b2b-architecture-guardian → multi-tenant-guardian → Skills do domínio → release-validator`.
Billing: `b2b2b-architecture-guardian → billing-payment-guardian → mercadopago-r96 → plan-transition-guardian`.
WhatsApp: `b2b2b-architecture-guardian → master-whatsapp-automation-engine → whatsapp-scheduling-engine`.

## 32. Entregáveis
Antes de implementar, produzir: hierarquia atual/alvo, tenant real, actors/roles, ownership, matriz de acesso, Direct vs Provider, billing, cascatas, suporte, migrations aditivas, módulos que NÃO alterar, riscos, fases e testes.

## 33. Critério de sucesso
A conversão só é bem-sucedida quando Providers administram sua carteira enquanto cada tenant continua usando as mesmas funcionalidades de antes, sem perda de dados, duplicação de motores, vazamento entre empresas ou regressão operacional. Provider é camada nova; Tenant continua sendo tenant. A operação saudável existente é patrimônio do sistema e deve ser preservada.
