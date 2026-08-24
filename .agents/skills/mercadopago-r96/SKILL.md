---
name: mercadopago-r96
description: Implementa e homologa Mercado Pago Checkout Pro no padrão R96 Atomic com checkout server-side, referência opaca, webhook assinado, Payment oficial, reconciliação e idempotência.
---
# Mercado Pago R96 Atomic
Audite e consolide o billing existente. Browser/success_url/payload do webhook não aprovam pagamento. Backend resolve tenant e preço, cria checkout reference, cria Checkout Pro, valida webhook, consulta Payment oficial, valida ref/tenant/plano/valor/moeda/status e aplica payment/subscription/entitlement atomicamente. Webhook, polling e reconcile usam a mesma lógica. Pending/rejected não liberam; replay não duplica; secrets somente backend; refund/chargeback preservam histórico. Combine com `plan-transition-guardian`.