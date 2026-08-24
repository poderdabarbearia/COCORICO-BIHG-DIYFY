---
name: plan-transition-guardian
description: Audita e protege upgrade, renovação, downgrade, trial, cancelamento, mudanças agendadas, feature gates e entitlements.
---
# Plan Transition Guardian
Plano válido financeiramente define os recursos. Upgrade só após Payment oficial aprovado; renovação preserva período futuro e replay não duplica; downgrade mantém recursos já pagos até o fim do ciclo e não concede plano pago gratuitamente; trial/cancelamento/refund/chargeback são fluxos distintos. Proteja tenant/role, backend feature gates, RLS/grants e atomicidade. Combine com `billing-payment-guardian` e `mercadopago-r96`.