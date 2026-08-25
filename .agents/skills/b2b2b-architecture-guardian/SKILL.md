---
name: b2b2b-architecture-guardian
description: Arquitetura para converter SaaS multiusuário/multiempresa em B2B2B adicionando Providers acima dos tenants sem quebrar módulos existentes.
version: 1.0
canonical_name: poder-b2b2b-architecture-guardian
---
# B2B2B Architecture Guardian

## Missão
Transformar qualquer SaaS em `PLATFORM/MASTER → PROVIDER/PARTNER/RESELLER/FRANCHISEE → TENANT/COMPANY/UNIT/BUSINESS → USERS/TEAM → OPERAÇÃO`, em qualquer nicho. Provider é camada nova acima do tenant e NÃO o substitui.

## Regra máxima
> NUNCA reescrever o núcleo operacional se o isolamento atual por tenant funciona. Auditar Auth, tenant, memberships, autorização, RLS, RPCs, Edge/server functions, Storage, billing, workers, webhooks, WhatsApp e módulos. Migração deve começar ADITIVA e preservar tudo saudável.

## Fundação
Preferir `provider`, `provider_user`, `tenant.provider_id nullable` ou equivalentes existentes. `provider_id = NULL` = cliente direto e deve conservar o fluxo antigo. Manter `Platform → tenants diretos` e `Platform → Providers → tenants da carteira`.

## Preservação
Se dados operacionais já usam tenant_id, resolver `registro → tenant → provider`; não espalhar provider_id sem forte necessidade de ledger/auditoria/histórico/performance. Não duplicar agenda, vendas, billing, WhatsApp, estoque, Auth, CRM ou relatórios. Regra: `reusar → adaptar → proteger`.

## Membership/ownership
`provider_user != tenant_user`. Provider não vira admin do tenant automaticamente. Separar created_by de provider_id. Suporte não cria membership temporária do tenant.

## Escopo server-side
Nunca confiar em provider_id/reseller_id/tenant_id/company_id do frontend como autoridade. Resolver `auth.uid() → membership → provider ativo → ownership do tenant → role → ação`. Se múltiplos Providers forem possíveis, exigir contexto ativo explícito; senão, constraint de vínculo único. Proibido escolher silenciosamente o primeiro vínculo.

## Autorização
Separar `has_tenant_access`, `has_provider_access`, `provider_owns_tenant`, `is_provider_admin`, `my_provider_context`. NÃO adicionar provider ownership automaticamente à função global de acesso do tenant. Provider, por padrão, não lê PII, financeiro, caixa, agenda sensível, não exclui dados e não vira tenant admin. Tenant continua operando como antes.

## Suporte
Usar sessão explícita, temporária, auditada, expirável e revogável: `support_session(provider_id, tenant_id, actor_user_id, actor_role, reason, status, started_at, expires_at, ended_at)` + server functions dedicadas com dados mínimos/redigidos. Nunca abrir RLS global para Provider. Escritas de suporte exigem RPC/função específica, role, confirmação, idempotência e auditoria.

## Billing
Separar billing_subject, billing_channel, payment_provider, payer, seller e beneficiary. Billing Provider != Billing Tenant. Revenue share exige ledger explícito (gross, fee, platform share, provider share/net, payment id, split verification). Nunca liberar acesso por redirect; autoridade é API/webhook oficial + operação idempotente. Usar billing-payment-guardian/mercadopago-r96 para detalhes.

## Direct/Partner e criação
provider_id NULL = tenant direto; preenchido = carteira. Self-service cria tenant direto. Provider cria tenant com provider resolvido no servidor. Master vincula/desvincula por operação privilegiada auditada. Nunca usar provider_id do formulário como autoridade.

## Cascatas
Definir explicitamente efeitos de Provider ativo/suspenso/cancelado em criação, tenants existentes, billing, WhatsApp e serviços. Evitar UPDATE em massa quando estado derivado permitir reversão segura.

## White-label e scopes
Preferir `Tenant override → Provider config → Platform default`. Recursos compartilhados usam ownership explícito (`scope_type + scope_id`, platform/provider/tenant) para WhatsApp, automações, filas, campanhas, knowledge, avisos, templates, integrações, branding e notificações. Workers/webhooks reconstroem scope server-side e não misturam Provider A/B.

## Auditoria
Registrar ator, role, provider, tenant, ação, recurso, origem, timestamp e before/after seguro quando necessário; nunca secrets/PII desnecessária.

## Fases
0 Auditoria. 1 Fundação aditiva. 2 Master Providers. 3 Painel Provider sem tocar telas operacionais. 4 Isolamento/segurança. 5 Suporte controlado. 6 Billing Provider. 7 White-label/Marketplace opcional. Após cada fase testar Auth, onboarding, dashboard, agenda, clientes, equipe, estoque, vendas, checkout, financeiro, relatórios, assinaturas, notificações, WhatsApp e configurações.

## Anti-patterns
Nunca confiar em provider_id client-side; transformar Provider em tenant admin; abrir has_tenant_access globalmente; espalhar provider_id; duplicar motores/Auth/tenant; copiar migrations cegamente; converter tenants sem rollback; confundir gateway com Provider; misturar billings; aceitar redirect como pagamento; selecionar primeiro Provider ambiguamente; membership temporária de suporte; RLS ampla; remover tenants diretos; quebrar funcionalidades existentes.

## Testes e Quality Gate
Provider A→B bloqueado; Provider A→Tenant B bloqueado; Tenant A→B bloqueado; Tenant→Provider bloqueado; Provider→tenant direto bloqueado. Cobrir CRUD, RPC, server/Edge, Storage, workers, webhooks, WhatsApp, billing, exports, logs, mobile, build/typecheck/testes. Painel Provider funcionar não basta: tenant antigo deve continuar funcional.

## Skills
`project-auditor → b2b2b-architecture-guardian → multi-tenant-guardian → supabase-guardian → safe-code-change → release-validator → deploy-forensics`.
Novo: `new-system-architect → b2b2b-architecture-guardian → multi-tenant-guardian → skills do domínio → release-validator`.

## Entregáveis
Antes do código: hierarquia atual/alvo, tenant real, actors/roles, ownership, matriz de acesso, Direct vs Provider, billing, cascatas, suporte, migrations aditivas, módulos que NÃO alterar, riscos, fases e regressão.

## Sucesso
Só está pronto quando Providers administram a carteira e cada tenant mantém as mesmas funcionalidades anteriores, sem perda de dados, duplicação de motores, vazamento ou regressão. A operação saudável existente é patrimônio do sistema.
