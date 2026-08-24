---
name: whatsapp-scheduling-engine
description: Motor conversacional WhatsApp V2 para atendimento, FAQ, suporte, agenda e grupos selecionados.
---
# WhatsApp Scheduling Engine V2
Um único motor: webhook→tenant/instance→dedup→policy direct/group→state→determinístico/LLM→knowledge/tools→ação→confirmação→commit→outbound. Dados reais são autoridade; FAQ preserva booking; ações são idempotentes e concurrency-safe; grupos allowlist; human mode bloqueia IA; sem retry cego de envio incerto.