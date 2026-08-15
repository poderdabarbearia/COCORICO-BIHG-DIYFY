---
name: whatsapp-scheduling-engine
description: Implementa, consolida e homologa motores de agendamento por WhatsApp com IA, agenda real, confirmação explícita, idempotência, fila/worker e isolamento multi-tenant. Use para implantar do zero ou corrigir tentativas parciais sem criar um segundo motor.
---

# WhatsApp Scheduling Engine

## Missão
Implementar ou consolidar agendamento, remarcação e cancelamento via WhatsApp conectado à agenda real do sistema. Leia `AGENTS.md` e audite o código atual antes de alterar qualquer coisa.

## Classificação inicial
Classifique o motor como `ABSENT`, `FRAGMENTED`, `PARTIAL`, `FUNCTIONAL_UNHOMOLOGATED` ou `HOMOLOGATED`.

Trace: `webhook → tenant → inbound → dedup → conversation → state → intent/tool → availability → confirmation → commit → outbound → queue/worker → provider`.

Use `IMPLEMENT_FROM_ZERO` somente quando o motor estiver ausente. Para qualquer tentativa existente, use `REPAIR_AND_CONSOLIDATE`: escolha um único caminho canônico, reaproveite peças válidas, corrija apenas o delta e nunca deixe dois webhooks, chatbots ou workers disputando a mesma mensagem.

## Referências
- Sistema Corte e Barba: base estrutural prioritária para inbound, fila/worker, estado conversacional, booking e contratos.
- ZappBarber: referência comparativa de hardening para deduplicação, concorrência, fast-path, horários completos e prevenção de ações duplicadas.
- ZappBarber não vira referência universal do PODER PLATFORM CORE por causa desta Skill.

## Arquitetura canônica
`Evolution/provider → webhook protegido → resolver tenant server-side → normalizar/persistir inbound → deduplicar → claim FIFO por conversa → carregar estado → fast-path determinístico ou LLM → tools/RPCs server-side → persistir novo estado → preparar outbound uma vez → fila/send → registrar resultado`.

## Invariantes
1. Tenant é resolvido no servidor pela instância/provider; nunca confiar em `tenant_id`, `clinic_id`, `company_id` ou `barbershop_id` vindo do payload público.
2. Webhook é transporte, não autoridade de negócio. Validar secret/assinatura, ignorar `fromMe`/grupos quando aplicável e persistir antes da IA.
3. Exactly-once lógico: inbound único por tenant+provider_message_id, um run por inbound, máximo um processing por conversa, no máximo uma resposta IA por inbound e business action idempotente.
4. IA não é autoridade da agenda. Serviço, preço, duração, profissional e slots vêm de banco/RPC real.
5. Persistir o estado antes de enviar a pergunta que depende desse estado.
6. Resposta antiga deve ser descartada se a conversa avançou (`context_version`, sequence ou equivalente).
7. Depois de booking/reschedule/cancel commitado, falha de log/estado/envio nunca pode repetir a ação de negócio.

## Conversa e interpretação
Persistir estado durável para intenção, serviço, profissional ou `any`, data/período, slots exibidos, slot escolhido, booking alvo, confirmação pendente e versão do contexto.

Criar fast-path determinístico para `sim/não`, opções numéricas, datas, hoje/amanhã, dias da semana, manhã/tarde/noite e horários como `16`, `16h`, `16:00`, `10h30`. Diferenciar `opção 13` de `13:00`. Na ambiguidade, perguntar; nunca adivinhar.

Perguntas informativas no meio do fluxo devem responder com dados reais sem destruir o estado de booking ativo.

## Disponibilidade
A única fonte da verdade deve ser server-side. Considerar serviço e duração, profissional compatível, jornada semanal, intervalos, bloqueios, exceções, timezone, passado, antecedência, janela máxima, step, bookings existentes e concorrência.

Não truncar silenciosamente a lista de horários. Se houver muitos slots, paginar conversacionalmente. Se o cliente pedir um horário não exibido, consultar novamente o conjunto real antes de negar.

## Confirmação e commit
Booking: `escolha → resumo → confirmação explícita → revalidar slot → commit`.
Remarcação e cancelamento seguem o mesmo padrão e só podem operar bookings do contato dentro do tenant atual.

O commit deve ser server-side, tenant-safe, idempotente e protegido contra corrida por transaction, constraint, unique, advisory lock ou equivalente. Se duas confirmações disputarem o mesmo slot, somente uma pode vencer.

## Fila, worker e retry
Use fila durável com estados equivalentes a `pending/processing/sent/error/skipped`, attempts, next attempt, lock/lease, provider id, correlation e context version.

- Falha antes do I/O do provider: retry seguro com backoff.
- Timeout após início do envio e entrega incerta: não reenviar cegamente.
- Provider confirmou envio e persistência local falhou: registrar `DEGRADED`; não reenviar.

Fast-path imediato e worker precisam compartilhar o mesmo claim.

## Handoff humano
Persistir modo `ai/human/paused/closed` ou equivalente. Ao transferir para humano, salvar o modo antes da mensagem de handoff e revalidá-lo imediatamente antes do outbound. IA e humano não podem responder em paralelo.

## Segurança
Webhook protegido; tenant server-side; service role somente backend; mutações não executáveis por `anon`; `SECURITY DEFINER` com `search_path` seguro e grants mínimos; RLS coerente; logs sem secrets/PII excessiva; prompt injection não altera regras de tenant/agenda.

## Testes mínimos
Validar: saudação; serviços/preços; profissional específico e `any`; horários completos; períodos; formatos de hora; opção × hora; passado; sem slots; bloqueios; slot ocupado entre oferta e confirmação; duas confirmações concorrentes; webhook duplicado; worker+webhook concorrentes; resposta obsoleta; retry; confirmação obrigatória; `sim` duplicado; FAQ no meio do fluxo; listar/remarcar/cancelar booking próprio; bloquear booking de outro contato/tenant; handoff humano; provider desconectado; restart de run preso; prompt injection; logs seguros.

Classifique cada teste como `PASSOU`, `FALHOU`, `NÃO EXECUTADO` ou `NÃO APLICÁVEL`.

## Fechamento
Nunca diga apenas “pronto”. Reporte separadamente: `CODE_IMPLEMENTED`, `MIGRATION_VERSIONED`, `MIGRATION_APPLIED`, `EDGE/ROUTE_DEPLOYED`, `SECRETS_CONFIGURED`, `PROVIDER_CONNECTED`, `WORKER_ACTIVE`, `CONTRACT_TESTS_PASSED`, `WHATSAPP_E2E_PASSED` e `PRODUCTION_HOMOLOGATED`.

Não declarar E2E/produção homologados sem conversa real no provider quando esse teste não ocorreu.
