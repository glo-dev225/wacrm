# Démo « Commande Express » : audit phase 0

> Audit en lecture seule, réalisé le 5 novembre 2026.
> Aucune donnée, aucun workflow, aucun webhook, aucune variable d'environnement et aucun déploiement n'ont été modifiés.
> Seul ce fichier a été ajouté, sur la branche `claude/new-session-0mqacp`.

**Légende :** ✅ vérifié · ⚠️ vérifié, avec un risque · ❓ non vérifiable depuis cette session, à confirmer par Glorieux.

---

## 1. Synthèse

1. **Le cœur technique existe déjà dans wacrm.** Accusé de réception rapide à Meta (`after()`), idempotence sur l'identifiant de message, envoi et réception de boutons interactifs, récupération des médias, assignation, passage de relais IA, rôles `agent` et `viewer`, temps réel Supabase. Il manque la logique métier (catalogue, calcul, état de commande), la transcription et le tableau de bord.
2. **Le principal risque n'est pas le code. C'est la cohabitation avec la production.**
   - L'instance wacrm actuelle se redéploie **automatiquement à chaque push sur `main`**.
   - La base Supabase de production devrait recevoir les migrations de démo.
   - Le webhook Meta est partagé avec n8n, et je ne connais pas son état.
3. **Recommandation : une instance de démo séparée** (option B, section 5) plutôt qu'un ajout dans l'instance de production. L'isolation est réelle, le retour arrière consiste à tout éteindre, et rien ne touche la base de production.
4. **Six décisions bloquent la phase 1** (section 9). La plus importante : quelle URL de callback est configurée aujourd'hui dans l'app Meta, et pour quels numéros.

---

## 2. Code : version et écarts

| Point | Constat | État |
|---|---|---|
| Dépôt | `glo-dev225/wacrm`, fork de `ArnasDon/wacrm`, version `0.8.0` | ✅ |
| Commits propres au fork | 2 : `409a861 fix(build)` (passage de `next.config.ts` à `.mjs` et ajout du script `build:hostinger` avec `--webpack`, pour la glibc trop ancienne de Hostinger) et son merge | ✅ |
| Retard sur l'amont | **163 commits**, dont **17 migrations** (`043` à `059`). Parmi elles, des correctifs de sécurité : `043_storage_scoped_select`, `044_internal_definer_grants`, `047_account_membership_integrity`, `055_authz_membership_holes`, ainsi que des corrections de concurrence dans les moteurs (`051`, `052`, `053`, `058`) | ⚠️ |
| Migrations locales | `001` à `042` | ✅ |
| Next.js | 16.3.5. `AGENTS.md` signale des changements d'API par rapport aux versions connues. L'avertissement « middleware → proxy » apparaît au build | ✅ |
| Langues de l'interface | `en`, `es`, `ko`, `pt`. **Pas de français.** `NEXT_PUBLIC_APP_LOCALE` n'est pas défini sur l'instance, donc l'interface est en anglais | ⚠️ |
| Licence | `LICENSE` MIT (« Copyright (c) 2026 Arnas Donauskas ») présente, à conserver | ✅ |
| Recherche de « SST » et « Secure Solutions » | Aucun résultat dans le dépôt | ✅ |
| Dépendances | Pas de `zod`. La validation des réponses du LLM se fera avec un validateur écrit à la main, sans nouvelle dépendance | ✅ |

**Conséquence :** il faut décider si la démo part du fork tel quel ou de l'amont à jour, avec le correctif Hostinger réappliqué (question Q5).

---

## 3. Infrastructure observée

### 3.1 Hébergement web Hostinger (wacrm)

| Point | Constat | État |
|---|---|---|
| Site | Un seul site Node.js : `darkorchid-snail-315197.hostingersite.com` (compte `u351321002`). `wacrm.gtpulse.tech` répond avec la page wacrm et le webhook (`{"error":"Missing verification parameters"}`), c'est donc le même site derrière le CDN Hostinger | ✅ |
| `gtpulse.tech` | Domaine parqué (« Parked Domain »), il ne sert pas wacrm | ✅ |
| Build | Node 20, `npm run build:hostinger`, dernier build `completed` le 17/09/2026 sur le commit `71ea648` | ✅ |
| **Déploiement automatique** | **Activé sur `glo-dev225/wacrm`, branche `main`.** Tout merge sur `main` met la production à jour | ⚠️ |
| Variables d'environnement | `NEXT_PUBLIC_SITE_URL`, `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `ENCRYPTION_KEY`, `META_APP_SECRET`. Valeurs masquées, non lues. Pas d'`AUTOMATION_CRON_SECRET`, donc les étapes « Attendre » des automatisations ne sont pas vidées par un cron | ✅ |
| **API des variables d'environnement** | L'opération Hostinger de mise à jour **remplace l'ensemble des variables** : une variable oubliée est supprimée. Je ne m'en servirai pas. Toute variable à ajouter sera renseignée par Glorieux dans hPanel | ⚠️ |
| Traitements longs | `after()` garde le travail en vie après la réponse sous `next start`, mais le gestionnaire de processus de l'hébergement mutualisé (mise en veille, arrêt des processus inactifs) n'est pas documenté. Vocal, transcription puis LLM, c'est 5 à 20 s par message. À mesurer avant de s'y fier | ❓ |
| glibc < 2.29 | SWC natif indisponible, build en WASM, compilation lente (40 s). Aucun module natif ne pourra être ajouté (par exemple `ffmpeg`) | ⚠️ |

### 3.2 VPS Hostinger (n8n, Hermès)

| Point | Constat | État |
|---|---|---|
| Machine | `srv1979887.hstgr.cloud`, KVM 2 (2 vCPU, 8 Go, 100 Go), Ubuntu 24.04 avec Docker et Traefik, en marche | ✅ |
| Projets Docker | `traefik`, `hermes-agent` (image `nousresearch/hermes-agent:latest`), `n8n-with-ai-assistant-kssb` (n8n, plus une sandbox avec 3 conteneurs actifs et 1 conteneur d'initialisation arrêté, ce qui est normal) | ✅ |
| n8n | `n8n.gtpulse.tech` pointe vers le VPS (187.7.25.114) | ✅ |
| Observation hors périmètre | Le port 32771 de n8n est publié sur `0.0.0.0`, donc n8n est joignable en HTTP direct, sans passer par Traefik. Par ailleurs, `:latest` sur Hermès et n8n veut dire qu'un redémarrage peut changer de version. Je n'y touche pas, je le signale | ⚠️ |
| Workflows n8n, configuration d'Hermès | Pas d'accès depuis cette session (ni clé d'API n8n, ni SSH). Je ne sais pas quel workflow reçoit les événements WhatsApp, ni pour quel numéro | ❓ |

### 3.3 Supabase

| Point | Constat | État |
|---|---|---|
| Projet utilisé par l'instance | La connexion Supabase de cette session n'expose aucun outil de lecture (ni liste de projets, ni SQL). Je n'ai donc vérifié ni l'état des migrations appliquées, ni l'existence d'une `whatsapp_config`, ni la présence de comptes | ❓ |
| Sauvegarde | Impossible à vérifier d'ici. Le plan (Free ou Pro) détermine si des sauvegardes automatiques et la restauration à un instant donné existent | ❓ |

### 3.4 Meta (WhatsApp Cloud API)

| Point | Constat | État |
|---|---|---|
| Webhook wacrm | `/api/whatsapp/webhook`, avec signature HMAC vérifiée par `META_APP_SECRET` (plusieurs secrets acceptés, séparés par des virgules) | ✅ |
| **Callback configuré dans l'app** | Inconnu. Une app Meta n'a **qu'une** URL de callback par défaut. Si n8n l'occupe, wacrm ne reçoit rien aujourd'hui, et l'inverse est aussi vrai | ❓ |
| Surcharge par numéro | Meta permet une URL de callback **propre à un numéro** (`POST /<PHONE_NUMBER_ID>` avec `webhook_configuration.override_callback_uri`). L'ordre de priorité est : numéro, puis WABA, puis app. Cela couvre `messages`. C'est la piste la moins intrusive : seul le numéro de test serait redirigé, n8n ne serait pas touché. Elle reste à approuver (règle 2) | ✅ (doc) |
| Limite des boutons | **20 caractères** par libellé, 3 boutons au maximum (doc Meta). wacrm applique déjà `INTERACTIVE_LIMITS.buttonTitleMaxLength = 20` | ✅ |
| Libellés du brief | Le plus long, « Moins de 10 cartons », fait 19 caractères. **Tous respectent la limite** | ✅ |
| Numéro de test Meta | Nom affiché imposé par Meta, non personnalisable (ce qui est conforme à la règle 5). Au plus **5 destinataires vérifiés**, ce qui sert aussi de garde-fou « téléphones de test uniquement » | ✅ (doc) |

---

## 4. Capacités réelles de wacrm, comparées aux besoins

| Besoin du brief | Existant (fichier) | Écart |
|---|---|---|
| Accusé rapide et traitement asynchrone | `webhook/route.ts` : réponse 200 immédiate, travail dans `after()`, `maxDuration = 60` | Aucun |
| Idempotence | Insertion `upsert … onConflict: conversation_id,message_id, ignoreDuplicates`. En cas de rejeu, tout le traitement aval (flows, automatisations, IA, webhooks) est sauté | Aucun, à condition de brancher la démo **après** ce point |
| Boutons interactifs (envoi) | `sendInteractiveButtons` (`meta-api.ts`), `engineSendInteractiveButtons` (`flows/meta-send.ts`), qui persistent aussi le message dans la conversation | Aucun |
| Boutons interactifs (réception) | `interactive.button_reply.id`, stocké dans `messages.interactive_reply_id` | Aucun |
| Récupération du vocal | `getMediaUrl` et `downloadMedia`, plus un miroir dans le bucket `chat-media` | Aucun |
| **Transcription** | Absente | **À écrire** |
| LLM | `src/lib/ai/` : OpenAI, Anthropic et fournisseurs compatibles OpenAI. Clé fournie par le compte, **chiffrée en AES-256-GCM** dans `ai_configs` | Sortie JSON validée par schéma **à écrire**. Le moteur actuel produit du texte libre |
| Catalogue, prix, total, numéro de commande | Absent | **À écrire** (code déterministe, sans LLM) |
| Assignation | `conversations.assigned_agent_id` | Aucun |
| Bot muet après reprise | `ai_autoreply_disabled` (verrou persistant) et arrêt de l'auto-réponse quand un humain est assigné | Réutilisable. Le bot de démo doit lire **les mêmes champs** |
| Relais vers un humain | `handoff.ts` : résumé et `handoff_agent_id` | Réutilisable |
| Notes, étiquettes, fiche contact | `contact_notes`, étiquettes, contacts | Aucun |
| Rôles | `owner > admin > agent > viewer`, RLS `is_account_member` | `viewer` correspond au compte direction en lecture seule. **Un agent voit toutes les conversations du compte**, pas seulement les siennes : c'est acceptable pour la démo, à signaler pour la suite |
| Notifications commerciales | `notifications` (migration 027) | L'alerte « encours » est à brancher dessus |
| Temps réel | Publication `supabase_realtime` sur `messages` et `conversations` | À ajouter pour `demo_snem_*` |
| Diffusions | Présentes | À masquer ou désactiver dans le compte de démo |
| Interface en français | **Absente** | Voir Q4 |

**Point d'intégration retenu :** un seul appel ajouté dans `processMessage`, **après** l'insertion idempotente et **avant** `dispatchInboundToFlows`. Il est actif uniquement si `account_id === SNEM_DEMO_ACCOUNT_ID` (variable d'environnement absente en production, donc le code est inerte). Toute la logique vit dans `src/lib/demo-snem/`. La divergence avec l'amont tient en environ 10 lignes dans `route.ts`.

---

## 5. Architecture : deux options

### Option A : dans l'instance actuelle (ce que le brief présuppose)

- Les tables `demo_snem_*` sont ajoutées à la base de production.
- Un compte wacrm dédié « Démo Commande Express » est relié au numéro de test.
- Le code est déployé sur `wacrm.gtpulse.tech`.

| Pour | Contre |
|---|---|
| Une seule instance | Migrations sur la base de production, et l'état réel des migrations (001 à 042) n'est pas vérifiable |
| Pas de nouvel hébergement | Déployer la démo, c'est déployer la production : merge sur `main` et build automatique |
| | Traitements longs sur l'hébergement mutualisé non garantis |
| | Retour arrière : revert, plus des migrations inverses sur la production |

### Option B (recommandée) : instance de démo séparée

- **Code :** cette branche, sans merge sur `main`.
- **Exécution :** un **nouveau** projet Docker Compose sur le VPS (le `Dockerfile` et `docker-compose.yml` existent déjà), derrière Traefik, sur un sous-domaine neutre, par exemple `demo-commande.gtpulse.tech`. C'est un ajout : les projets existants ne sont pas touchés.
- **Données :** un **nouveau projet Supabase**, gratuit, dédié. Toutes les migrations wacrm et `demo_snem_*` y sont appliquées. Zéro migration sur la base de production.
- **Meta :** le numéro de test est surchargé au niveau du numéro vers `https://<sous-domaine-démo>/api/whatsapp/webhook`. Le callback de l'app et le flux n8n restent inchangés. Nécessite l'accord de Glorieux.

| Pour | Contre |
|---|---|
| Isolation réelle : base, processus et domaine | Un projet Supabase et un conteneur en plus à gérer |
| Pas de contrainte glibc ni de mise en veille, traitements longs possibles | Le VPS héberge déjà n8n et Hermès (8 Go de RAM). wacrm consomme environ 300 à 500 Mo |
| Retour arrière : arrêter le conteneur, supprimer la surcharge Meta. La production n'a jamais bougé | Un ajout sur le VPS, à approuver |
| Partir de l'amont à jour (correctifs de sécurité) sans toucher à la production | |

> **Angle mort à lever :** « réaliser la démo sur mon installation wacrm » et « ne rien casser, isoler » tirent dans des sens opposés tant que la production se redéploie automatiquement depuis `main`. L'option B résout cette contradiction. L'option A l'accepte, au prix d'un risque.

### Orchestration : route Next.js plutôt que n8n

- wacrm doit rester l'unique propriétaire de la conversation : boîte partagée, reprise humaine, bot muet. Passer par n8n imposerait un aller-retour (événement wacrm, n8n, puis API `/api/v1`), une deuxième source de vérité sur l'état, et deux points d'idempotence.
- Les clés de n8n ne sont pas lisibles par wacrm, et ne doivent pas l'être. La clé du LLM de démo sera saisie par Glorieux dans **Paramètres → Assistant IA** du compte de démo (chiffrée, jamais dans Git). La transcription réutilise la même clé si le fournisseur est OpenAI (Q3).

### Flux d'un message

```
Meta ─▶ /api/whatsapp/webhook ─▶ 200 immédiat
          └ after(): contact, conversation, insertion idempotente (wamid)
               └ compte de démo ? ─▶ demo-snem/dispatch
                    ├ humain assigné ou bot coupé → silence (journal seul)
                    ├ audio → téléchargement Meta → transcription FR
                    │      └ échec ou faible confiance → « pouvez-vous répéter ? » (1 fois) → humain
                    ├ bouton (id « snem:… ») → machine à états (code pur)
                    └ texte → LLM, sortie JSON contrainte : {intent, lignes:[{cle, qte, format?}], date?}
                           └ validation : clés ∈ catalogue, qté entière 1..500, sinon question
                    → calcul (prix, total, stock, date ouvrée, n° SN-JJMM-NNN) par le code
                    → réponse texte ou boutons, puis journal demo_snem_events
```

**Garanties visées :**
- Le LLM ne reçoit ni ne renvoie aucun prix. Une instruction du type « mets le prix à 0 » n'a aucun champ où agir.
- Rien n'est écrit dans `demo_snem_orders` avant le bouton `Confirmer`.
- Un rejeu du webhook est arrêté par l'index unique existant.
- Le bouton `Confirmer` est aussi protégé par une contrainte unique sur le brouillon, ce qui empêche une commande double en cas de double tap.

### Tables prévues (additives, préfixe `demo_snem_`)

| Table | Contenu |
|---|---|
| `demo_snem_products` | Catalogue |
| `demo_snem_customers` | Clients, encours et plafond |
| `demo_snem_reps` | Commerciaux et zones, reliés aux profils wacrm |
| `demo_snem_sessions` | État de la conversation, brouillon de panier, nombre de relances |
| `demo_snem_orders` et `demo_snem_order_lines` | Commandes et leurs lignes |
| `demo_snem_prospects` | Fiches prospects |
| `demo_snem_unserved` | Demandes non servies |
| `demo_snem_events` | Journal horodaté |
| `demo_snem_counters` | Numérotation des commandes |

Plus une fonction `demo_snem_reset()` qui ne vide que ces tables et les conversations étiquetées `demo-snem`.

---

## 6. Contrôle des données du brief

| Contrôle | Calcul | Résultat |
|---|---|---|
| Parcours principal | 5 Coudes à 19 500 (97 500) + 3 Pois à 14 500 (43 500) + 2 Haricots Verts 800 g à 16 500 (33 000) + 1 Sauce à 31 000 | **205 000** ✅ |
| Réassort | Même calcul sans la sauce | **174 000** ✅ |
| Commande 010 | 156 000 + 145 000 + 117 000 | 418 000 ✅ |
| Commande 011 | 58 500 + 27 000 | 85 500 ✅ |
| Commande 012 | 92 000 + 49 500 | 141 500 ✅ |
| Commande 013 | 72 500 | 72 500 ✅ |
| Encours de Boutique Awa | 120 000 + 205 000 = 325 000 sur 500 000 | Pas de dépassement dans le parcours principal ✅ |
| Libellés de boutons | Le plus long fait 19 caractères | ≤ 20 ✅ |
| Fuseau `Africa/Abidjan` | UTC+0, sans heure d'été | Le calcul « après la fermeture » est simple ✅ |

**Incohérences ou zones floues relevées :**

1. **Ordre des scénarios 1 et 2.** Si Awa passe la commande 014 (scénario 1) puis écrit « comme la dernière fois », sa dernière commande est la 014 (205 000), pas celle du 22 octobre (174 000). Faut-il prendre la dernière commande *validée antérieure au jour*, utiliser un autre client ou téléphone pour le scénario 2, ou remettre à zéro entre les deux ? (Q6)
2. **Assignation et silence du bot.** La conversation est assignée à M. Koné dès la confirmation (étape 5). La règle wacrm actuelle (humain assigné, donc bot muet) rendrait le bot silencieux dès l'étape 5. Le brief attend pourtant que l'étape 7 provoque un « transmet et cesse de répondre ». Ma proposition : après confirmation, le bot ne répond plus qu'aux événements de la commande (validation). Tout message libre déclenche un accusé « je transmets à M. Koné », puis le bot est coupé (`ai_autoreply_disabled = true`). Cela reste cohérent avec la règle existante. (Q6)
3. **« Livrer lundi ».** Nous sommes jeudi 5 novembre 2026. « Lundi » est donc le 9 novembre, un jour ouvré futur, accepté. La règle « jour ouvré suivant » ne joue que pour un samedi après 12 h ou un dimanche. Les jours fériés ivoiriens ne sont pas gérés (hors périmètre, à confirmer).
4. **La maquette `SNEM_Commande_Express_maquette.html` n'a pas été fournie** dans cette session. J'en ai besoin pour la phase 4 (tableau de bord).

---

## 7. Risques

| # | Risque | Probabilité | Impact | Parade |
|---|---|---|---|---|
| R1 | Un merge sur `main` déploie la démo en production | Moyenne | Fort | Option B. En option A, ne jamais merger sans accord |
| R2 | Migration de démo sur la base de production sans sauvegarde vérifiée | Moyenne | Fort | Option B. En option A, sauvegarde vérifiée (`pg_dump` et restauration testée) avant la phase 1 |
| R3 | Modifier le callback de l'app Meta coupe n8n | Moyenne | Fort | Surcharge **par numéro** uniquement, après accord |
| R4 | Transcription peu fiable (accents, bruit, mots locaux comme « macaroni Panzani-là ») | Forte | Moyen | Indices de vocabulaire du catalogue passés au modèle de transcription, confirmation systématique par récapitulatif, relance unique puis humain, mesure sur ≥ 10 vocaux |
| R5 | Le LLM invente une ligne ou une quantité | Moyenne | Moyen | Clés limitées au catalogue, validation stricte, récapitulatif obligatoire avant toute écriture |
| R6 | Fenêtre de 24 h de Meta : la notification « Validée » arrive plus de 24 h après le dernier message client | Faible en démo | Moyen | En démo, la validation suit de près. En production, il faudrait un modèle de message approuvé |
| R7 | Interface commerciale en anglais pendant l'atelier SNEM | Certaine | Moyen (image) | Q4 |
| R8 | Le fork a 17 migrations de retard, dont des correctifs d'autorisation | Certaine | Moyen | Q5 |
| R9 | Coûts LLM et transcription incontrôlés | Faible | Faible | Limite de tours par session, journal `ai_usage_log`, coupure par variable d'environnement |
| R10 | Charge supplémentaire sur le VPS (option B) | Faible | Moyen | Limite mémoire Docker (`mem_limit`), surveillance |

---

## 8. Plan de retour arrière

**Option B :**
1. Supprimer la surcharge Meta du numéro de test (`override_callback_uri` vide). Le numéro revient au callback de l'app.
2. `docker compose down` dans le **seul** dossier du projet de démo. Traefik, n8n et Hermès ne sont pas touchés.
3. Mettre en pause ou supprimer le projet Supabase de démo, après accord.
4. La production n'a jamais changé, il n'y a rien à restaurer.

**Option A :**
1. Retirer `SNEM_DEMO_ACCOUNT_ID` dans hPanel. Le code devient inerte sans redéploiement de code, seulement un redémarrage.
2. Revert du merge sur `main`, ce qui déclenche un build automatique.
3. Migration inverse `DROP TABLE demo_snem_* CASCADE`. Les tables sont exclusivement de démo, et le compte de démo est supprimé par `account_id`.
4. Restauration de la sauvegarde seulement si l'étape 3 échoue.

Ce plan sera testé en phase 1 (critère 10) et son résultat consigné.

---

## 9. Décisions attendues avant la phase 1

| # | Question | Ma recommandation |
|---|---|---|
| **Q1** | Option A (instance actuelle) ou **option B** (instance de démo séparée : VPS, nouveau projet Supabase, sous-domaine) ? | B |
| **Q2** | Webhook Meta : quelle URL de callback est configurée aujourd'hui dans l'app, et pour quels numéros n8n répond-il ? Le numéro de test appartient-il à la même app, et donc au même `META_APP_SECRET` ? M'autorises-tu une **surcharge au niveau du seul numéro de test** ? | Surcharge par numéro |
| **Q3** | LLM et transcription : quel fournisseur pour la démo ? Tu saisiras toi-même la clé dans Paramètres → Assistant IA. Ma proposition : OpenAI, une seule clé pour la compréhension et la transcription (`gpt-4o-transcribe`, à défaut `whisper-1`), plafond de dépense fixé côté fournisseur | OpenAI, une seule clé |
| **Q4** | Interface en français : ajouter `messages/fr.json` (plus de 2 000 chaînes, ajout pur, réutilisable en amont), ou franciser seulement `/demo/snem` et les messages WhatsApp ? | `fr.json` si l'atelier montre la boîte partagée ; sinon page de démo seule |
| **Q5** | Base de code : fork actuel, ou amont à jour avec le correctif Hostinger réappliqué ? (Avec l'option B, ce choix ne touche pas la production) | Amont à jour |
| **Q6** | Règles métier : (a) scénario 2 sur le même téléphone qu'Awa, et quelle commande compte comme « la dernière fois » ? (b) Valides-tu la règle d'assignation et de silence du bot du §6, point 2 ? (c) Jours fériés ignorés ? | (a) dernière commande antérieure au jour (b) oui (c) oui |

**À fournir également (sans aucun secret dans la conversation) :**
- La maquette HTML.
- Les numéros des téléphones de test : 5 au maximum avec le numéro de test Meta.
- La date cible.
- Avec l'option B : le nom du sous-domaine. Les variables d'environnement seront renseignées par toi dans un `.env` sur le VPS, dont je fournirai la liste des clés.
- Avec l'option A : l'identifiant du projet Supabase de production, et la preuve d'une sauvegarde restaurable.

---

## 10. Ce que cet audit n'a pas pu vérifier

- L'état réel de la base Supabase : migrations appliquées, `whatsapp_config`, comptes existants.
- Les workflows n8n et la configuration d'Hermès (pas d'accès).
- La configuration de l'app Meta : callback, abonnements, numéros.
- Le comportement de `after()` sur l'hébergement mutualisé quand un traitement dure plus de 10 s.
- La disponibilité et le coût actuels de `gpt-4o-transcribe` en français ivoirien. Ce sera mesuré en phase 3, et aucun taux ne sera annoncé avant.
