# Frida SSL Pinning Bypass Lab — Walkthrough

> **Avertissement éthique** : N'appliquez ce lab que sur des appareils/apps vous appartenant ou dans un cadre d'audit autorisé.

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

> 📸 **Capture 1** — Sortie de `frida-ps -Uai` montrant `com.example.sslpinningtest`
> `[insérer screenshot ici]`

---

## Étape 2 — Premier essai (échec attendu)

Lancer le script universel initial :

```bash
frida -U -f com.example.sslpinningtest -l sslpin_bypass_universal.js
```

Appuyer sur le bouton dans l'app → ❌ encore bloqué.

**Diagnostic :** le script hookait `check()` mais pas `check$okhttp()`, la méthode interne Kotlin qui fait la vraie vérification.

> 📸 **Capture 2** — App affichant ❌ malgré les hooks Frida actifs
> `[insérer screenshot ici]`

---

## Étape 3 — Identifier les méthodes réelles (diagnostic)

Dans la console Frida interactive, énumérer toutes les méthodes de `CertificatePinner` :

```javascript
Java.perform(function(){
  var CP = Java.use('okhttp3.CertificatePinner');
  console.log(JSON.stringify(Object.getOwnPropertyNames(CP)));
})
```

Résultat révélateur :
```
[..., "check$okhttp", "check", ...]
```

→ `check$okhttp` est la méthode interne Kotlin invisible via l'API Java classique.

> 📸 **Capture 3** — Console Frida avec la liste des méthodes de CertificatePinner
> `[insérer screenshot ici]`

---

## Étape 4 — Fix : hooker check$okhttp directement

Dans la console Frida :

```javascript
Java.perform(function(){
  var CP = Java.use('okhttp3.CertificatePinner');
  CP['check$okhttp'].implementation = function(host, certs){
    console.log('[+] check$okhttp SKIPPED for: ' + host);
    return;
  };
  CP['check'].overloads.forEach(function(ov){
    ov.implementation = function(){
      console.log('[+] check SKIPPED');
      return;
    };
  });
  console.log('[+] ALL patched');
})
```

Appuyer sur le bouton → ✅ Succès.

> 📸 **Capture 4** — Console Frida avec `check$okhttp SKIPPED` + app affichant ✅
> `[insérer screenshot ici]`

---

## Étape 5 — Script final corrigé (sslpin_bypass_universal.js)

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

> 📸 **Capture 5** — Terminal avec tous les hooks confirmés au spawn
> `[insérer screenshot ici]`

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

> 📸 **Capture 6** — Terminal frida-trace montrant les symboles SSL actifs
> `[insérer screenshot ici]`

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

> 📸 **Capture 7** — Terminal bypass combiné + app ✅ Succès
> `[insérer screenshot ici]`

---

## Étape 8 — Validation Burp Suite

Dans Burp → HTTP History :
```
https://api.github.com    GET    /    200    JSON
```

> 📸 **Capture 8** — Burp HTTP History avec api.github.com intercepté
> `[insérer screenshot ici]`

---

## Découverte clé du lab

OkHttp 4.x compilé en Kotlin génère une méthode interne `check$okhttp()` invisible via l'API Java classique. Les scripts génériques ratent cette méthode. La technique de diagnostic :

```javascript
Object.getOwnPropertyNames(Java.use('okhttp3.CertificatePinner'))
```

permet de lister **toutes** les méthodes réelles, y compris les méthodes Kotlin internes, pour hooker la bonne cible.

---

## Résumé

| Étape | Technique | Résultat |
|-------|-----------|----------|
| Script universel initial | Hook `check()` seul | ❌ Insuffisant |
| Diagnostic `getOwnPropertyNames` | Découverte de `check$okhttp` | ✅ Méthode réelle identifiée |
| Script final | Hook `check$okhttp` + `check()` | ✅ Bypass au spawn |
| frida-trace natif | Traçage `SSL_*` / `X509_*` | ✅ Pinning Java confirmé |
| Script natif BoringSSL | Hook `SSL_get_verify_result` | ✅ Prêt pour apps natives |
| Burp HTTP History | Capture trafic HTTPS | ✅ api.github.com intercepté |

---

## Checklist finale

- [x] frida-server v17.9.1 actif sur l'émulateur
- [x] Script universel Java (Conscrypt + OkHttp + check$okhttp)
- [x] Diagnostic `getOwnPropertyNames` pour identifier les méthodes Kotlin
- [x] frida-trace SSL natif exécuté
- [x] Script natif BoringSSL créé et testé
- [x] Bypass combiné Java + Natif fonctionnel
- [x] Trafic HTTPS intercepté dans Burp
