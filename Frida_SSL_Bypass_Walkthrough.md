# Frida SSL Pinning Bypass Lab — Walkthrough


---

## Environnement

| Élément | Valeur |
|--------|--------|
| PC Hôte | Windows 11 |
| Émulateur | Genymotion Android 11 (x86_64) |
| IP Hôte | 192.168.56.1 |
| Proxy | Burp Suite (port 8080) |
| Frida | v17.9.1 |
| App cible | com.example.sslpinningtest (OkHttp 4.12.0) |

---

## Objectif

Bypasser le SSL pinning d'une app Android avec **Frida directement** (sans Objection), via des scripts JavaScript ciblant les couches Java et natives.

---

## Prérequis

- frida-server v17.9.1 actif sur l'émulateur
- App SSLPinningTest installée
- Burp Suite configuré comme proxy (192.168.56.1:8080)

---

## Étape 1 — Démarrer frida-server

```bash
adb shell "/data/local/tmp/frida-server &"
```

Vérifier :
```bash
frida-ps -Uai
```

---

<img width="1292" height="192" alt="image" src="https://github.com/user-attachments/assets/c888214e-634b-4d0a-8db8-a26ce8ded66f" />

---

<img width="1092" height="477" alt="image" src="https://github.com/user-attachments/assets/c094a5e2-4aee-449b-a53b-77fa45b8f473" />
`

---

## Étape 2 — test de l'application


Appuyer sur le bouton dans l'app → ❌ encore bloqué.


---

<img width="1600" height="819" alt="image" src="https://github.com/user-attachments/assets/baf5e97e-ed91-43b9-99f9-02a6d7056cb4" />


---


## Étape 3 — Script  (sslpin_bypass_universal.js)

Le fix `check$okhttp` a été intégré dans le script final pour fonctionner **directement au spawn** :

```bash
frida -U -f com.example.sslpinningtest -l sslpin_bypass_universal.js
```

Résultat attendu au démarrage :
```
[+] SSL bypass: SSLContext.init patched
[+] SSL bypass: TrustManagerImpl patched
[+] SSL bypass: check$okhttp patched
[+] SSL bypass: OkHttp CertificatePinner fully patched
[+] Universal SSL pinning bypass installed
```

---

<img width="1600" height="795" alt="image" src="https://github.com/user-attachments/assets/5122f8ec-3402-46f7-b943-c50bf74e8b01" />

---

<img width="1600" height="731" alt="image" src="https://github.com/user-attachments/assets/5094a35c-fede-4997-b90b-1c8a8ad3b364" />

---

<img width="1600" height="719" alt="image" src="https://github.com/user-attachments/assets/06ef3a5a-0067-4e55-89df-29c896d03506" />

---

## Étape 6 — Traçage natif avec frida-trace

Pour identifier les symboles SSL natifs utilisés par l'app :

```bash
frida-trace -U -f com.example.sslpinningtest -i "SSL_*" -i "X509_*"
```

Symboles clés observés :
```
SSL_CTX_new()            → création du contexte SSL
SSL_set_custom_verify()  → callback de vérification custom
SSL_do_handshake()       → handshake TLS
SSL_get0_peer_certificates() → récupération du certificat serveur
SSL_shutdown()           → fermeture propre
```

**Conclusion :** `SSL_get_verify_result` absent → pinning 100% Java (OkHttp), pas natif.

---

<img width="1600" height="729" alt="image" src="https://github.com/user-attachments/assets/6e3176fb-2792-4a7d-b513-cedfd6bc20a2" />

---

<img width="1600" height="798" alt="image" src="https://github.com/user-attachments/assets/d85c3af6-588a-463a-8c8c-489431e718ef" />

---
## Étape 7 — Script natif BoringSSL (sslpin_bypass_native.js)

Créé pour les apps avec pinning natif. Deux hooks :

- `SSL_get_verify_result` → force le retour `X509_V_OK` (0)
- `SSL_set_custom_verify` → neutralise le callback custom

Lancement combiné Java + Natif :

```bash
frida -U -f com.example.sslpinningtest -l sslpin_bypass_universal.js -l sslpin_bypass_native.js
```

Résultat sur notre app :
```
[-] Hook failed for SSL_get_verify_result : not a function  ← normal (inline/optimisé)
[-] SSL_set_custom_verify hook failed: not a function        ← normal
[+] Native SSL bypass installed
[+] SSL bypass: check$okhttp SKIPPED for: api.github.com
✅ Succès
```

> Les hooks natifs échouent proprement (symboles non interceptables sur cet émulateur). Le bypass Java prend le relais.

---

<img width="1600" height="833" alt="image" src="https://github.com/user-attachments/assets/9e8ba53f-af15-47a1-83bb-18e6d67b353e" />

---

<img width="1600" height="801" alt="image" src="https://github.com/user-attachments/assets/c725b320-b619-4625-9b86-4c3543738f2c" />

---

## Étape 8 — Validation Burp Suite

Dans Burp → HTTP History :
```
https://api.github.com    GET    /    200    JSON
```

---

<img width="1600" height="818" alt="image" src="https://github.com/user-attachments/assets/38cf38f5-1f61-42a9-beb9-5c4a54153011" />

---


## Résumé

| Étape | Technique | Résultat |
|-------|-----------|----------|
| Script universel initial | Hook `check()` seul |  Insuffisant |
| Diagnostic `getOwnPropertyNames` | Découverte de `check$okhttp` |  Méthode réelle identifiée |
| Script final | Hook `check$okhttp` + `check()` |  Bypass au spawn |
| frida-trace natif | Traçage `SSL_*` / `X509_*` |  Pinning Java confirmé |
| Script natif BoringSSL | Hook `SSL_get_verify_result` |  Prêt pour apps natives |
| Burp HTTP History | Capture trafic HTTPS |  api.github.com intercepté |

---

## Checklist finale

- [x] frida-server v17.9.1 actif sur l'émulateur
- [x] Script universel Java (Conscrypt + OkHttp + check$okhttp)
- [x] Diagnostic `getOwnPropertyNames` pour identifier les méthodes Kotlin
- [x] frida-trace SSL natif exécuté
- [x] Script natif BoringSSL créé et testé
- [x] Bypass combiné Java + Natif fonctionnel
- [x] Trafic HTTPS intercepté dans Burp


## Auteur
**H-oubane**
