---
name: whatsapp-pairing-code
description: Adiciona ao motor WhatsApp já existente a opção de conectar por código de pareamento, mantendo QR Code, provider, instância canônica e isolamento multiempresa.
---
# WhatsApp Pairing Code

## Missão
Reger exclusivamente a extensão de um motor WhatsApp que já conecta por QR Code para também permitir **Conectar por código**. Não rege IA, atendimento, agendamento, campanhas, lembretes ou automações após a conexão.

## Regra zero — não criar segundo motor
Antes de alterar qualquer coisa, mapear `UI → backend → adapter Evolution → tabela/linha canônica → status → webhook`. Se o QR já existe, ESTENDER esse motor. Proibido criar `whatsapp_v2`, provider paralelo, tabela paralela, segundo webhook ou segunda fonte de verdade apenas para pairing code.

## Resultado esperado
A mesma instância deve aceitar duas estratégias de autenticação: **QR Code OU código de pareamento pelo telefone**. O QR existente deve continuar funcionando e permanecer como fallback.

## Fluxo canônico
`UI → telefone → auth server-side → tenant autorizado → linha canônica → estado real do provider → normalização → criação/reuso seguro da sessão → pairingCode → UI → provider confirma connected`.
Gerar código, receber HTTP 200 ou criar instância NÃO significa conectado. `connected` só pode vir do provider/webhook autenticado.

## Contrato Evolution comprovado
Quando o projeto usar Evolution API 2.3.7 + WHATSAPP-BAILEYS, preservar o contrato comprovado no Corte e Barba SaaS: `POST /instance/create` com `instanceName`, `qrcode: true`, `integration: "WHATSAPP-BAILEYS"` e `number` normalizado. Extrair `pairingCode` da resposta real. Não inventar endpoint alternativo sem confirmar a versão do provider.

## Telefone
Normalizar novamente no servidor: somente dígitos; Brasil aceita com ou sem 55 e deve produzir o formato esperado pelo provider. Rejeitar número inválido antes da chamada externa. Não confiar somente no frontend.

## Pairing code é efêmero
É proibido persistir ou registrar o código em PostgreSQL/Supabase, logs, console, analytics, localStorage, sessionStorage, audit log, histórico de chat ou tabela de mensagens. Exibir somente durante a tentativa ativa e descartar depois. Nunca criar coluna `pairing_code`.

## Secrets
Evolution URL/key, service role e webhook secret ficam somente server-side. Nunca aceitar `instance_name`, provider secret ou API key vindos do navegador. Logs devem sanitizar `pairingCode`, `pairing_code`, QR/base64, `code` longo, `apikey`, `authorization`, `token`, `secret` e `password`.

## Multiempresa
Autenticar usuário, autorizar empresa/tenant e carregar a instância no servidor. Nunca escolher “a primeira empresa do usuário”. Operações de conectar/reconectar devem respeitar owner/admin conforme regra do produto. O browser não decide a instância remota.

## Uma linha canônica
Pairing code não cria segunda identidade de negócio. Reutilizar a linha/tabela atual (ex.: `whatsapp_instances`). Mesmo se o nome da sessão remota precisar mudar, a empresa continua com uma única linha canônica.

## Rotação segura da sessão remota
Se Evolution/Baileys bloquear reutilização do nome com `already in use`, 403/409 conhecido ou sessão antiga presa, é permitido rotacionar SOMENTE o identificador remoto. Use base opaca + nonce curto aleatório; não use telefone, email, CPF ou UUID puro. Preserve a linha canônica.

Ordem segura: (1) preservar sessão anterior; (2) criar candidata; (3) obter pairing code; (4) configurar webhook necessário; (5) promover a nova `instance_name` na mesma linha com compare-and-swap quando houver concorrência; (6) só então limpar a sessão antiga quando seguro. Se falhar antes da promoção, remover candidata best-effort e manter a anterior.

## Sessão já conectada
Consultar estado real antes de gerar novo código. Sessão saudável `connected` não pode ser destruída automaticamente; exigir desconexão explícita. Estado desconhecido não vira `disconnected` por conveniência.

## Concorrência
Proteger duplo clique, duas abas, QR × código simultâneo e retries paralelos. Usar in-flight/cooldown e, quando aplicável, compare-and-swap na promoção. Retry não pode criar várias instâncias órfãs.

## Webhook e consumidores
Pairing é apenas outro método de conectar o mesmo WhatsApp. Se houver rotação, garantir que status, QR, webhook inbound, envio manual, IA/outbound, workers, diagnóstico e desconexão usem o `instance_name` salvo na fonte canônica, nunca um nome antigo recalculado por slug. Não criar webhook especial para pairing.

## Extração segura
Centralizar no adapter a extração de QR e pairing code. Não devolver payload bruto, HTML, base64 ou QR como se fosse código. Uma resposta com QR + pairing deve servir cada valor ao fluxo correto.

## Status/UI
Estados recomendados: `disconnected | connecting | connected | disconnecting/syncing | unknown/error`. A UI mostra QR e **Conectar por código** lado a lado/na mesma área, inclusive mobile. Ao escolher código: pedir telefone, validar, gerar, exibir de forma copiável, mostrar instruções do WhatsApp e acompanhar status real.

## WhatsApp pessoal e Business
Não limitar artificialmente ao WhatsApp Business. Permitir WhatsApp normal e Business quando suportados pelo provider/app.

## Erros e fallback
Tratar telefone inválido, API indisponível, 401, 403, 404, 409, timeout, nome em uso, pairing ausente, já conectado, concorrência e estado desconhecido. Não classificar todo 403 como API key inválida. Se não houver código válido, não inventar/reutilizar código antigo; manter QR como alternativa.

## Banco
Antes de migration, auditar schema/RLS/constraints. Em regra, pairing não exige tabela nova nem migration. Desejável: uma linha por empresa, `instance_name` único, RLS ativo, sem escrita sensível direta pelo cliente e sem coluna de pairing code.

## Testes obrigatórios
- QR continua gerando e conectando.
- Telefone BR com/sem 55 e telefone inválido.
- Código real aparece e não é persistido/logado.
- WhatsApp normal/Business quando suportado.
- Gerar código não marca connected.
- Provider conectado atualiza status real.
- Sessão conectada não é derrubada.
- Duplo clique, duas abas e QR × pairing não criam sessões concorrentes.
- API key/service role não chegam ao frontend.
- Tenant A não controla instância do tenant B.
- Após conexão por código: status, webhook, envio e desconexão continuam funcionando.

## Homologação
Build ou botão visível não bastam. Homologar fisicamente: abrir WhatsApp real → Aparelhos conectados → conectar com número → informar código → provider confirmar connected → UI refletir connected → validar webhook/envio/desconexão → confirmar QR ainda funcional.

## Anti-padrões proibidos
Segundo motor/tabela/provider; remover QR; salvar pairing code; expor secrets; aceitar instance_name arbitrário do browser; marcar connected após HTTP 200; destruir sessão saudável; retry infinito; desabilitar RLS; escolher primeira empresa; migration desnecessária; endpoint de outra versão da Evolution sem verificar contrato.

## Ordem de implementação
1. Auditar QR atual e adapter/provider.
2. Identificar tenant e linha canônica.
3. Confirmar versão/contrato Evolution.
4. Preservar QR.
5. Adicionar CTA + telefone.
6. Implementar ação server-side autenticada e normalização.
7. Consultar estado real e gerar código no mesmo motor.
8. Rotacionar sessão remota somente se necessário.
9. Não persistir código; sanitizar logs.
10. Acompanhar status até confirmação do provider.
11. Revisar consumidores de `instance_name`.
12. Testar QR, pairing físico e regressão pós-conexão.

## Critério de sucesso
O sistema passa de **QR** para **QR OU código**, usando o mesmo motor, provider, empresa, fonte de verdade, linha canônica, webhook e status, sem persistir credenciais efêmeras, sem expor secrets e sem regressão do fluxo existente.