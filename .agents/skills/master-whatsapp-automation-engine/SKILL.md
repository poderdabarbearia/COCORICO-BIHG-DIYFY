---
name: master-whatsapp-automation-engine
description: Orquestra o WhatsApp do Painel Master para lifecycle, onboarding, cobrança, suporte e memória operacional da plataforma, reutilizando instância, filas, billing e knowledge existentes sem criar motores paralelos.
---
# Master WhatsApp Automation Engine

## Missão
Criar, consolidar ou reparar o motor de automações do WhatsApp do Painel Master de um SaaS da PODER PLATFORM.

Quatro pilares: (1) instância WhatsApp Master; (2) lifecycle/automações; (3) fila/worker/health; (4) suporte inteligente com memória do sistema.

## Regra zero
Auditar antes de criar. Classificar o motor como ABSENT, FRAGMENTED, PARTIAL, FUNCTIONAL_UNHOMOLOGATED ou HOMOLOGATED. Reusar contratos existentes; nunca criar provider, worker, chatbot, fila ou tabela paralela quando já houver equivalente saudável.

## Fronteiras
- Campanhas/broadcast: `whatsapp-campaign-engine`.
- Agendamento/remarcação/cancelamento: `whatsapp-scheduling-engine`.
- Pagamento: `billing-payment-guardian` + `mercadopago-r96`.
- Transição de plano: `plan-transition-guardian`.
- Esta Skill orquestra o Control Plane Master.

## Arquitetura canônica
Outbound: `domain event → automation rule → condition recheck → template/AI → master queue → atomic claim/lease → provider → delivery log → audit/health`.
Inbound: `Master WhatsApp → authenticated webhook → dedupe → identity resolver → conversation → context → Master Knowledge + live tools → AI → outbound queue → provider → persistence`.

## Instância Master
- Instância exclusiva da plataforma, isolada dos tenants.
- Provider é autoridade do status de conexão.
- QR apenas quando necessário.
- Reconectar não derruba sessão saudável.
- No máximo uma instância ativa por escopo `platform_master`, salvo arquitetura explícita.
- Webhook Master com secret próprio.
- Secrets apenas server-side/Vault.

## Eventos mínimos
Identidade/onboarding: `user.created`, `tenant.created`, `onboarding.started`, `onboarding.completed`, `whatsapp.connected`, `whatsapp.disconnected`, `user.inactive`.
Trial/assinatura: `trial.started`, `trial.ending`, `trial.expired`, `subscription.activated`, `subscription.renewed`, `subscription.expiring`, `subscription.overdue`, `subscription.suspended`, `subscription.reactivated`, `subscription.cancelled`.
Pagamento: `payment.pending`, `payment.approved`, `payment.failed`, `payment.refunded`, `payment.chargeback`.
Automação consome eventos; nunca decide sozinha estado financeiro.

## Automações padrão
`welcome_account`, `welcome_tenant`, `onboarding_whatsapp_missing`, `onboarding_core_incomplete`, `trial_started`, `trial_ending`, `trial_expired`, `subscription_expiring_7d`, `subscription_expiring_3d`, `subscription_expiring_1d`, `subscription_due_today`, `subscription_overdue_1d`, `subscription_overdue_3d`, `subscription_suspended`, `subscription_reactivated`, `payment_pending`, `payment_approved`.
Offsets devem ser configuráveis.
Antes de enviar reminder temporal, revalidar o estado real; se a pendência foi resolvida, marcar `skipped/cancelled`.

## Idempotência
Toda causa deve gerar no máximo um job lógico. Usar chave equivalente a `<automation_key>:<subject_type>:<subject_id>:<event_id|period|due_date>`.
Retries reutilizam a mesma identidade; novo ciclo legítimo gera nova identidade. Replay de webhook não duplica confirmação. `user.created` e `tenant.created` não podem gerar boas-vindas semanticamente duplicadas.

## Fila/worker
Estados: `pending → processing → sent|skipped|retrying|failed|dead_letter`.
Obrigatório: claim atômico (`FOR UPDATE SKIP LOCKED` ou lease equivalente), batch configurável, backoff, stale-lock recovery, dead-letter, pacing, reprocessamento controlado, sem retry cego de envio incerto, desconexão não consome tentativa, processamento independente de tela aberta.

## Controles independentes
`master_sends_enabled`, `master_automations_enabled`, `master_ai_enabled`, `scheduler_enabled`. Um switch não deve desligar responsabilidades não relacionadas.

## Health
Expor sem secrets: scheduler configurado/ativo, última execução/sucesso/falha/bloqueio, capturados/enviados/skipped/retrying/failed/dead-letter, backlog, idade do pendente mais antigo, estado real da instância, webhook e heartbeat. `configurado != funcionando`.

## Memória operacional
Três camadas:
1. Knowledge curado: onboarding, módulos, funcionalidades, WhatsApp, IA, planos, políticas, FAQ, troubleshooting homologado, links e releases.
2. Live tools server-side para dados mutáveis: plano, trial, vencimento, assinatura, pagamentos, entitlements, WhatsApp, onboarding, links/configurações não sensíveis e health apropriado.
3. Memória da conversa: mensagens, resumo, assunto, dados confirmados, estado `ai|waiting_human|human`, última interação e identidade resolvida.
Nunca guardar secrets/tokens/senhas.
Autoridade: `live tool oficial > banco/config oficial > knowledge homologado > histórico > inferência LLM`.

## Política da IA Master
Responder curto e útil; buscar determinístico primeiro; usar RAG para instruções e live tools para estado atual; não inventar; oferecer handoff; nunca revelar outro tenant; nunca prometer ação não executada. Não usar `whatsapp-scheduling-engine` para dúvidas do SaaS.

## Identidade/privacidade
Perguntas públicas podem usar knowledge pública. Para dados de conta, resolver identidade server-side, aplicar autorização e limitar escopo. Telefone não é tenant authority. Sem fallback cross-tenant. Alterações sensíveis exigem app/sessão/handoff apropriado.

## Handoff
Suportar `ai`, `waiting_human`, `human` e retomada explícita da IA. Ao pedir humano ou atingir baixa confiança/ação sensível, parar IA, preservar histórico, notificar Inbox Master e auditar sender.

## Segurança Supabase
Aplicar `supabase-guardian` + `multi-tenant-guardian`: RLS, anon sem acesso privado, usuário comum sem INSERT em fila Master, super admin server-side, service role restrita, SECURITY DEFINER com search_path fixo/autorização correta, secrets no Vault, logs sanitizados e audit log.

## Portabilidade
Mapear conceitos ao schema existente: tenant/company/store/clinic; owner/user/profile; master instance; automation catalog; queue; delivery log; support conversations/messages; knowledge; settings; heartbeat; event log/outbox. Se existir equivalente válido, estender o existente.

## Bugs que deve impedir
Master misturado com tenant; falso conectado; reconexão destrutiva; duas instâncias Master; boas-vindas duplicadas; evento de billing nunca emitido; cron no código mas ausente no ambiente; scheduler sem heartbeat; fila dependente da UI; crons concorrentes; retry cego; lock eterno; tentativa consumida desconectado; reminder obsoleto; knowledge vazia; artigo antigo vencendo dado vivo; RAG cross-tenant; secret em UI/log; SECURITY DEFINER inseguro; migration histórica quebrando RLS; suporte Master usando motor vertical de agenda.

## Workflow
1. Auditoria.
2. Escolher caminho canônico: reutilizar/corrigir/consolidar/desativar legado/criar delta.
3. Homologar instância Master.
4. Conectar Domain Events/lifecycle.
5. Homologar fila/worker/scheduler/heartbeat.
6. Homologar knowledge + live tools + memória + inbound + handoff.
7. UI Master para conexão, switches, catálogo, fila, erros, inbox, knowledge e health.
8. Quality Gate.

## Testes obrigatórios
Instância real/QR/refresh/reconnect/disconnect/isolamento; boas-vindas sem duplicação; cron sem browser; concorrência sem duplicação; retry/backoff/stale lock/heartbeat; reminders 7/3/1/dia/overdue revalidados; replay webhook sem duplicar; knowledge ativa/desativada; live tool vence artigo antigo; sem invenção/cross-tenant; handoff para humano; RLS/grants/secrets/webhook/SECURITY DEFINER seguros.

## Homologação
Só marcar HOMOLOGATED com instância real isolada, webhook autenticado, eventos da fonte correta, idempotência, fila concorrente segura, scheduler real, heartbeat recente, retries/locks tratados, reminders revalidados, knowledge populada, live tools, inbound, handoff, segurança, Quality Gate verde e produção na mesma versão. Código presente não prova runtime.
