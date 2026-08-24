---
name: billing-payment-guardian
description: Audita, protege e consolida billing, Mercado Pago, assinaturas, webhooks, reconciliação e entitlements.
---
# Billing Payment Guardian
Audite antes de alterar e não crie segundo motor. Browser/success_url/query não são autoridade financeira. Backend resolve tenant, plano, preço e moeda; webhook/reconcile consultam Payment oficial e só então aplicam operação atômica/idempotente em payment/subscription/entitlement. Pending/rejected não liberam; replay não duplica; refund/chargeback preservam histórico; secrets somente backend. Use `mercadopago-r96` para Checkout Pro e `plan-transition-guardian` para mudanças de plano.