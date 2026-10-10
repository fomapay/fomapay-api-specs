# fomapay-api-specs

Source de vérité des SDKs FomaPay (dépôt public).

- `openapi.json` — spécification OpenAPI 3 exportée du Core (`GET /docs-json`). Les SDKs utilisent l'API publique **`/v1/*`**.
- Ce document — le **contrat commun** que chaque SDK respecte, quel que soit le langage.

Pour régénérer la spécification depuis un backend local :

```bash
curl -s http://localhost:8080/docs-json > openapi.json
```

> Les modèles des SDKs sont écrits à la main et alignés sur cette spécification ; ils ne sont pas (encore) générés automatiquement.

## Contrat commun des SDKs

### Clés

| Clé | Où | Peut |
| --- | --- | --- |
| `pk_test_…` / `pk_live_…` | navigateur, application mobile | créer un paiement, le régler, suivre son statut |
| `sk_test_…` / `sk_live_…` | serveur uniquement | tout le reste : lister, solde, retraits |

Un SDK serveur refuse une clé qui ne commence pas par `sk_` ; un SDK client refuse une clé qui ne commence pas par `pk_`.

### Format standard d'une transaction

```json
{
  "amount": 1000.00,
  "currency": "CDF",
  "orderId": "CMD-2026-001",
  "customer": { "name": "Jean Marc", "phone": "+243810000000", "email": "jean@example.com" },
  "paymentModes": ["fomapay_direct", "mobile_money"],
  "callbackUrl": "https://marchand.cd/api/callback",
  "metadata": { "custom_field": "valeur" }
}
```

`currency` vaut `USD` ou `CDF`. `callbackUrl` est conservée et renvoyée avec le paiement ; les notifications serveur passent
par les **webhooks** configurés dans FomaPay Business.

### Dual-mode

- `fomapay_direct` — le client paie depuis son portefeuille FomaPay (QR code). Disponible dès le **Niveau 0**.
- `mobile_money` — M-Pesa, Orange Money, Airtel Money. Réservé aux boutiques vérifiées (**Niveau 1+**) ; toujours ouvert en sandbox.

### Ressources et méthodes

| Méthode | Appel HTTP | Clé |
| --- | --- | --- |
| `payments.create(params)` | `POST /v1/payments` | pk / sk |
| `payments.retrieve(id ou référence)` | `GET /v1/payments/{id}` | pk / sk |
| `payments.payWithMobileMoney(id, operator, phone)` | `POST /v1/payments/{id}/mobile-money` | pk / sk |
| `payments.simulate(id, outcome)` — sandbox | `POST /v1/payments/{id}/simulate` | pk / sk de test |
| `payments.list(limit, status)` | `GET /v1/payments` | sk |
| `payouts.create(params)` | `POST /v1/payouts` | sk |
| `balance.retrieve()` | `GET /v1/balance` | sk |
| `store.retrieve()` | `GET /v1/store` | pk / sk |

`transactions` est un alias de `payments` dans tous les SDKs.

### Erreurs

Réponse d'erreur de l'API : `{ "error": { "code": "ERROR_…", "message": "…", "details": {…} } }`.

| Type | Quand |
| --- | --- |
| `ApiError` | l'API a répondu avec une erreur (`code`, `message`, `status`, `details`) |
| `NetworkError` | pas de réponse : connexion impossible (`ERROR_NETWORK`) ou délai dépassé (`ERROR_TIMEOUT`) |
| `SignatureError` | signature de webhook invalide ou expirée (`ERROR_INVALID_SIGNATURE`) |

### Délai et nouvelles tentatives

- Délai par requête : **30 s** par défaut.
- **2 nouvelles tentatives** par défaut, avec attente exponentielle (0,5 s, 1 s, … plafonnée à 8 s) et aléa.
- Rejouées : erreurs réseau, délais dépassés et réponses 5xx — **uniquement si la requête est rejouable sans risque**
  (`GET`, ou `POST` portant un en-tête `Idempotency-Key`). Les réponses 429 sont toujours rejouées, en respectant `Retry-After`.
- `payments.create` envoie toujours une `Idempotency-Key` (fournie ou générée) : une coupure réseau ne crée jamais deux paiements.
- `payouts.create` n'est **jamais** rejoué automatiquement après une erreur réseau ou 5xx : un retrait ne doit pas partir deux fois.

### Webhooks

En-tête `X-FomaPay-Signature: t=<horodatage>,v1=<HMAC-SHA256(secret, "<horodatage>.<corps brut>")>` (hexadécimal).

`Webhook.verify(corpsBrut, entête, secret, tolérance = 300 s)` :
1. compare les signatures en **temps constant** ;
2. refuse un horodatage hors tolérance (anti-rejeu) ;
3. s'applique au **corps brut** de la requête, jamais à un objet re-sérialisé.

`Webhook.constructEvent(...)` vérifie puis renvoie l'événement, ou lève `SignatureError`.

Événements : `payment.succeeded`, `payment.failed`, `payment.expired`, `payout.succeeded`, `payout.failed`, `webhook.test`.

### Sandbox

Clés `*_test_*` : aucune somme réelle. Numéros Mobile Money de test — finissant par `0001` : refus ; `0002` : jamais validé ;
tout autre : succès après ~3 s. `payments.simulate` force l'issue sans téléphone.

### Tests d'intégration des SDKs

Chaque SDK embarque des tests d'intégration contre le sandbox, activés par trois variables d'environnement :
`FOMAPAY_API_URL`, `FOMAPAY_TEST_SECRET_KEY` (SDKs serveur) et `FOMAPAY_TEST_PUBLIC_KEY` (SDKs client).
