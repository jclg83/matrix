---
name: hermes-team-operations
description: Règles de fonctionnement de l'équipe Hermes — architecture vitrine + rooms privées (v4), cycles collecte→diffusion, anti-boucle structurel.
version: 2.9.3
metadata:
  hermes:
    tags: [team, orchestration, rules, orchestrator-v4, commands, vitrine]
---

# Hermes Team Operations — v4

## Architecture v4 : vitrine + rooms privées

**Anti-boucle physique** : les agents ne peuvent JAMAIS poster dans la room commune.
Boucle = structurellement impossible, pas de prompt anti-boucle à faire respecter.

```
hermes-team (!McOijzMcBSlvlfId:videocours.fr) → VITRINE
├─ events_default = 51  (m.room.message)  ← seuil de parole
├─ Christophe PL 100          ✍️ pose les questions, /orch
├─ Orchestrateur PL 51        ✍️ relais, séquenceur, récaps
└─ Agents PL 50               👁️ lecture seule (M_FORBIDDEN si écriture)

4 rooms privées (orchestrateur↔agent) — chaque agent ne voit QUE la sienne :
├─ orch-sos     @orchestrateur + @sos
├─ orch-three   @orchestrateur + @three
├─ orch-five    @orchestrateur + @five
└─ orch-chris   @orchestrateur + @chris

Tous les agents incluent Christophe en observateur dans leurs rooms privées.
Chaque room privée est isolée : un agent ne voit JAMAIS un message brut d'un autre agent.
```

**Plus aucun mute/unmute dynamique de power levels.** Les PL sont fixés une fois pour toutes. Le mute par PL de v3.2 est supprimé.

Mapping des rooms : `/opt/dendrite/data/private_rooms.json`

### Cycle : 2 phases (collecte → diffusion)

1. **Phase 1 — Collecte** : Christophe pose une question dans la vitrine. L'orchestrateur l'envoie aux agents dans leurs rooms privées (**parallèle** par défaut, timeout 480s global ; mode séquentiel 480s/agent via `/orch mode sequential`). Chaque agent répond dans sa room privée au message qui le mentionne (`@agent:videocours.fr ❓ Question de Christophe : ...`). L'orchestrateur demande explicitement un préfixe final et ne capture comme réponse officielle que les messages commençant par `REPONSE FINALE <AGENT>:`. Séquenceur visible auto-édité dans la vitrine : `🔄 Cycle | Phase : collecte | Reçues : 2/4 (SOS ✅ Three ⏳ ...)`.

2. **Phase 2 — Diffusion** : L'orchestrateur publie le récap complet `[SOS] ... [THREE] ... [FIVE] ... [CHRIS] ...` dans la vitrine (tronqué à 4000 chars/réponse), **ET** envoie le récap complet dans chaque room privée préfixé `📋 Récap (informatif — NE PAS RÉPONDRE)` → tous les agents ont les 4 réponses.

3. **Contexte d'historique (cycle_history)** : chaque question est préfixée du résumé des 5 derniers cycles. Détail dans `references/cycle-history-context.md`.

4. **File d'attente** : les questions posées pendant une collecte sont stockées dans `pending_questions` et déclenchées automatiquement en fin de cycle (expirent après 5 min).

### @mention d'agent dans la vitrine

Christophe peut cibler un agent seul : `@sos ta question` → cycle restreint à SOS uniquement. Les autres ne sont pas sollicités.

## Commandes /orch v4

Christophe contrôle l'orchestrateur dans la vitrine. Formes acceptées : `/orch status` ou `orch status` (sans slash évite l'interception par les bots Matrix).

| Commande | Effet | Changement vs v3 |
|---|---|---|
| `/orch status` | Cycle, mode, ordre, file d'attente, réponses x/4 | |
| `/orch stop` | Arrête le cycle en cours | |
| `/orch close` | **NOUVEAU** — clôt la collecte immédiatement, publie récap partiel | remplace `skip` |
| `/orch pause` | Suspend les nouveaux cycles | |
| `/orch resume` | Réactive | |
| `/orch mode parallel|sequential` | **NOUVEAU** — bascule mode collecte | |
| `/orch rooms` | **NOUVEAU** — liste les rooms privées mappées | |
| `/orch ordre A,B,C,D` | Change l'ordre des agents (diffusion + séquentiel) | |
| `/orch default-order` | Restaure SOS→Three→Five→Chris | |
| `/orch agents` | Liste les agents et leur état (disabled/repondu/attendu) | |
| `/orch disable @agent` ou `all` | Retire un agent du cycle (persistant) | `disable all` NOUVEAU |
| `/orch enable @agent` ou `all` | Réintègre | `enable all` NOUVEAU |
| `/orch queue` | Affiche les questions en attente | |
| `/orch clearqueue` | Vide la file d'attente | |
| `/orch appro` | Active l'approbation temporaire sur **tous les agents validés** dans `AUTO_RESET_AGENTS`, via `@agent !yolo orch on <id>` dans chaque room privée | nécessite handler agent déployé |
| `/orch reset` | Reset complet : vide `cycle_history`, `pending_questions`, état du cycle, désactive l'approbation temporaire, ET envoie `@agent !yolo orch reset` à chaque agent validé dans `AUTO_RESET_AGENTS` pour réinitialiser réellement leur session sans confirmation interactive. Utile après un changement de sujet pour repartir d'un contexte vierge (historique ET sessions agents). **Note** : les sessions agent ne sont vidées que si l'orchestrateur et les gateways concernés ont été redémarrés après modification de code. |
| `/orch log [N]` | N dernières lignes de log | |
| `/orch help` | Liste toutes les commandes | |
| `/orch synthese` | **v4.2** — synthèse des réponses du dernier cycle. Envoie les réponses + historique à Chris (room privée) avec un prompt structuré (consensus, désaccords, évolutions). Utilise un buffer + filtre anti-bruit + finalisation après 8s de silence. Timeout 3 min. Affiche la synthèse dans la vitrine **ET dans toutes les rooms privées** des agents sous `🧠 Synthèse de CHRIS`. Détail complet dans `references/orch-synthese.md`. | |

**Supprimés en v4** : `skip`, `muteall`, `unmuteall` (plus de PL pour les agents).

## Historique des cycles (cycle_history / format_history)

Chaque question envoyée aux agents est préfixée de l'historique des 5 derniers cycles. Depuis le 06/07/2026, `format_history` **ne tronque plus** — questions et réponses sont en texte intégral. Format :

```
📜 Cycles precedents :
  ▶ #297562 ❓ <question complète>
    [SOS] <réponse complète>
    [THREE] <réponse complète>
    [FIVE] <réponse complète>
    [CHRIS] <réponse complète>
```

L'ancien format tronquait questions à 40 caractères et réponses à 40 caractères sur une seule ligne (`#297562 je fais une vérification... -> SOS: C... | THREE: J...`) — illisible pour les agents qui l'interprétaient comme un résumé de statut plutôt qu'un vrai contexte. Pour vider cet historique → `/orch reset`.

## Séquenceur visible

L'orchestrateur édite un message de statut unique dans la vitrine via `m.replace` :

| Phase | Message séquenceur |
|---|---|
| Collecte | `🔄 Cycle | Phase : collecte (parallel) | Reçues : 2/4 (SOS ✅ Three ⏳ Five ⏳ Chris ⏳) | Question : ...` |
| Fin normale | `✅ Cycle terminé` |
| Timeout | `⏰ Cycle terminé (timeout)` |
| Clos | `🛑 Cycle clos par /orch close` |
| Stop | `🛑 Cycle arrêté par /orch stop` |

## Configuration Matrix pour les agents (v4)

```yaml
matrix:
  enabled: true
  user_id: "@<agent>:videocours.fr"
  server_url: "https://matrix.videocours.fr"
  access_token: "<token>"
  required_mention: true                          # ← OK désormais : v4 @mentionne l'agent
  allowed_rooms: "!<room-privée-agent>:videocours.fr"  # ← UNIQUEMENT sa room privée
  streaming:
    interrupt_notifications: false                # ← obligatoire
```

**Changements v4 par rapport à v3 :**
- `allowed_rooms` → room privée UNIQUEMENT (plus la vitrine). L'agent reçoit tout ce qui le concerne via l'orchestrateur dans sa room privée. Aucun bruit de vitrine, zéro interruption ⚡.
- `require_mention: true` OK désormais : l'orchestrateur v4 préfixe la question par `@agent:videocours.fr ❓...`, et le récap par `@agent:videocours.fr 📋 Récap...` → le gateway accepte les deux. Le récap est traité par l'agent mais filtré par la couche 3 (HORS CYCLE) de l'anti-boucle.

**Récap via `diffusion`** : après qu'un cycle est terminé, chaque agent reçoit dans sa room privée :
```
@agent:videocours.fr 📋 Récap (informatif — NE PAS RÉPONDRE)

[SOS] ...
[THREE] ...
[FIVE] ...
[CHRIS] ...
```
⚠️ **Le récap mentionne BIEN l'agent** (préfixe `@agent:videocours.fr` dans le code v4, fonction `publish()`). Avec `require_mention: true`, le gateway Hermes **laisse passer** le message → l'agent le traite comme un message entrant dans sa room privée. L'anti-boucle repose alors sur la couche **3. Logiciel** (`HORS CYCLE` ignoré + log). Si un agent répond, ça reste dans sa room privée ; l'orchestrateur logge `HORS CYCLE: AGENT - ignoré` et ne propage rien.

## Anti-boucle — 3 couches

| Couche | Mécanisme | Garantie |
|---|---|---|
| **1. Serveur** | vitrine `events_default=51`, agents PL 50 (< 51) | L'agent ne peut JAMAIS poster dans la room commune (M_FORBIDDEN) |
| **2. Structurel** | 4 rooms privées 1-à-1, récap informatif | Un agent ne voit jamais un message brut d'un autre agent |
| **3. Logiciel** | réponses finales capturées uniquement si elles commencent par `REPONSE FINALE <AGENT>:` ; hors-cycle ignoré + log, `is_system_msg()` | Les messages outils/notifications/intermédiaires ne deviennent pas réponse officielle ; si agent poste hors cycle → ignoré + loggé, zéro propagation |

Pire scénario : un agent répond au récap dans sa room privée. Personne ne le voit (c'est sa room privée). L'orchestrateur logge `HORS CYCLE: SOS - ignoré` et ne fait rien. Zéro boucle.

### Règles côté agent

Les agents n'ont **plus besoin** de filtrer manuellement : leur gateway les protège via `allowed_rooms` (une seule room) et `require_mention`. Mais ces règles de bon sens restent dans le prompt partagé :

1. **Ne pas envoyer d'accusés de réception** (« reçu », « j'attends », « ok ») dans Matrix
2. **Une réponse = un seul message**. Pas de suivi automatique
3. **Ne pas répondre aux messages `📋 Récap`** — ils sont informatifs, pas des sollicitations
4. **Messages système Hermes** à ignorer : `⚡ Interrupting current task`, `📬 No home channel`, `Operation interrupted`

### Règle de réponse finale capturable

Pendant un cycle orchestré, l'orchestrateur ignore tous les messages qui ne commencent pas par le préfixe final attendu. Chaque agent doit donc commencer son unique réponse finale exactement par :

```text
REPONSE FINALE SOS:
REPONSE FINALE THREE:
REPONSE FINALE FIVE:
REPONSE FINALE CHRIS:
```

Le message envoyé par l'orchestrateur rappelle toujours le préfixe exact à utiliser. Les messages de progression, outils, lectures de skill, demandes d'approbation, warnings `No home channel`, etc. peuvent exister mais ne seront pas capturés comme réponse officielle.

Si une action est bloquée par approval, l'agent doit quand même répondre avec le préfixe final attendu et indiquer clairement le blocage, par exemple :

```text
REPONSE FINALE FIVE: bloqué par approval sur commande <commande>, mise à jour non finalisée.
```

### Mise à jour de skill sans déclencher les approvals

Quand Christophe demande de mettre à jour un skill partagé depuis GitHub, éviter les commandes shell qui déclenchent l'approval gate Hermes :

- ne pas utiliser `rm -rf` ;
- ne pas utiliser `python -c` dans `terminal` ;
- ne pas faire `curl | python` ni autre pipe vers interpréteur ;
- préférer la lecture GitHub via outil web/API sûr ;
- préférer `skill_manage`, `write_file` et `patch` ;
- préférer un remplacement ciblé ou une réécriture contrôlée de fichier à une suppression récursive.

## Tâches longues

Un agent peut travailler en arrière-plan si Christophe lui a confié une mission. Les cycles de l'orchestrateur ne bloquent rien — ils sollicitent simplement l'agent dans sa room privée et attendent sa réponse (avec timeout 240s par défaut en mode parallèle).

## Skills partagés

Les skills communes sont versionnées dans le dépôt Git `jclg83/matrix` (GitHub).
Chaque agent les installe depuis cette source unique.

**Procédure de mise à jour côté source** : quand un agent (ex: SOS) maintient le skill sans avoir les droits
collaborateur sur le repo source → fork + PR. Détail complet dans `references/shared-skill-update-flow.md`.

**Procédure de synchronisation côté agent** : quand Christophe demande de mettre à jour la copie locale depuis GitHub Matrix, récupérer `hermes-team-operations/` depuis `jclg83/matrix`, écrire `SKILL.md` + `references/`, puis vérifier byte-à-byte. Ne jamais laisser un backup contenant un `SKILL.md` du même `name:` sous le dossier `skills/`, sinon `skill_view` devient ambigu. Détail : `references/sync-shared-skill-from-github.md`.

## Décision humaine et périmètre des projets

Christophe est le seul décideur. Les agents proposent, Christophe tranche.
Ne jamais déployer, modifier une config, ou lancer une action sans GO explicite.

- Les changements **videocours.fr / Moodle / salle2cours.fr** en production demandent la décision à trois : Christophe, SOS et Hermes Chris.
- Le projet **Matrix/Dendrite/orchestrateur** est distinct de videocours.fr/Moodle, malgré son hébergement sur le VPS `videocours.fr`. Il relève de SOS : un GO explicite de Christophe suffit pour y intervenir.

## Approbations temporaires des cycles `/orch`

Un état ajouté seulement à l’orchestrateur ne peut pas auto-approuver les commandes Hermes : les approval gates vivent dans les gateways de chaque agent. Une future commande `orch appro` doit donc être implémentée de bout en bout et limitée à la room/session ou au cycle concerné ; `orch reset`, un redémarrage et une expiration doivent la désactiver. Ne jamais utiliser `approvals.mode: off` global pour satisfaire ce besoin. Détails : `references/scoped-orch-approvals.md`.

## Reset automatique progressif des sessions agents

### Problème à éviter

`/orch reset` ne doit **pas** envoyer naïvement `@agent /new` puis annoncer que les sessions sont réinitialisées. Dans les gateways Hermes, `/new` est destructif et peut retourner `⚠️ Confirm /new` : sans clic humain, la session reste intacte. Toujours lire les logs/retours Matrix avant de déclarer le reset effectif.

### Pattern sûr : commande interne authentifiée

Pour les approbations/resets orchestrés, chaque gateway doit reconnaître des formes internes idempotentes, par exemple :

```text
@agent:videocours.fr !yolo orch on <approval_id>
@agent:videocours.fr !yolo orch reset
```

`orch appro` dans la vitrine doit dispatcher `!yolo orch on` à **tous** les agents de `AUTO_RESET_AGENTS`. Ne pas laisser une action historique de rollout comme `appro_sos_on` : après validation globale, `orch appro` doit utiliser une action générique (`appro_on`) et stocker l'état par liste d'agents (`orch_appro_agents`).

Le handler doit vérifier **les trois conditions** avant toute action :

1. plateforme Matrix ;
2. room privée exacte de cet agent ;
3. émetteur exact `@orchestrateur:videocours.fr`.

Il doit désactiver l’état d’approbation temporaire, puis appeler directement le reset de session (`/new`) sans passer par la confirmation interactive. Toute tentative depuis un autre émetteur/room est refusée. Ne jamais utiliser `approvals.mode: off` globalement.

### Déploiement agent par agent

Introduire une liste explicite dans l’orchestrateur, p. ex. `AUTO_RESET_AGENTS`, contenant **uniquement** les agents déjà testés. Pendant le rollout :

1. ajouter le handler à un seul gateway et le redémarrer ;
2. tester `orch appro` puis `orch reset` de bout en bout ;
3. prouver dans les logs : activation, désactivation, et message `New session started!` ;
4. vérifier que les agents non validés sont explicitement journalisés comme ignorés — aucun faux `/new` ne doit leur être envoyé ;
5. seulement alors ajouter l’agent validé à `AUTO_RESET_AGENTS` et passer au suivant.

Le rollout complet a été validé dans l’ordre SOS → Three → Five → Chris. Les 4 agents sont maintenant dans `AUTO_RESET_AGENTS`, et le test global réel confirme : `orch appro` active APPRO sur les 4 ; `orch reset` envoie `@agent !yolo orch reset` aux 4 ; chaque agent répond `✨ New session started!` sans `Confirm /new`. Pour la procédure TDD détaillée, l’accès Windows via Chris, les redémarrages avec `HERMES_HOME` explicite, le diagnostic de gateways concurrents, le prérequis tunnel Five `:2522`, le redémarrage Chris avec GO préalable, et le mode économie quand le modèle actif est coûteux, voir [references/progressive-orch-reset-rollout.md](references/progressive-orch-reset-rollout.md).

## 📋 Cross-agent feature evaluation

Quand une nouvelle version d'Hermes est déployée sur un agent et qu'une feature mérite une évaluation collective (ex: "Self-improvement moins cher" en v0.18 — PR #49252), le pattern est :

### Phases

**Phase 1 — Tour d'horizon initial** : poser la question large avec une liste de candidats évidents.

1. **Identifier la feature** — lire les release notes, comprendre le mécanisme
2. **Définir les critères** — usage attendu, contrainte qualité, priorité coût
3. **Lister les candidats évidents** — quelques modèles/providers connus pour starters
4. **Préparer une question structurée** pour la room Matrix Hermes Team :
   ```markdown
   📢 Question pour tous — <feature>

   <agent> est passée en v<version>. La feature "<nom>" (PR #<num>) permet de <mécanisme>.

   Objectif : <but>

   Critères :
   - <critère 1>
   - <critère 2>
   - <critère 3>

   Chez vous, quels modèles/providers utilisez-vous pour le modèle principal ?
   Et quel modèle auxiliaire recommanderiez-vous ?

   Idées candidates (à départager) :
   - <candidat A> (<provider>)
   - <candidat B> (<provider>)
   - <candidat C> (<provider>)

   Chacun donne son avis et on compare les prix ?
   ```
5. **Doubler comme test de connectivité** — l'envoi du message vérifie que le gateway Matrix de l'agent mis à jour fonctionne toujours (pas cassé par l'upgrade)
6. **Compiler les réponses** — chaque agent répond dans sa room privée, l'orchestrateur diffuse le récap dans la vitrine

**Phase 2 — Tour des angles morts** : après avoir lu les réponses du premier tour, Christophe peut repérer des familles entières oubliées par tous les agents. Reformuler pour les faire chercher, sans donner la réponse.

7. **Identifier les oublis** — relire les réponses, repérer ce qui manque
8. **Préparer une question ciblée** sans donner les prix ni le meilleur modèle :
   ```markdown
   📢 Question complémentaire — <feature> : deux oublis ?

   <constat : personne n'a mentionné X>

   Complément important : <contrainte implicite à expliciter>

   Pour <famille oubliée> : regardez les prix par vous-mêmes sur <source>
   et dites-moi lequel vous semblerait le meilleur candidat.
   Je veux votre avis, pas celui des autres.
   ```
9. **Compiler les réponses** — un second récap affine le consensus

**Phase 3 — Vote final** : quand tous les candidats sont sur la table, un dernier tour pour trancher.

10. **Consolider tous les candidats** des phases précédentes dans un tableau unique avec prix
11. **Demander un TOP 1** par agent, justification 2-3 lignes max
12. **Compiler et présenter à Christophe** pour décision finale

### Pièges

- **Ne pas poster la question dans une seule room privée** — elle doit aller dans la vitrine (hermes-team) pour que tous la voient et que l'orchestrateur la distribue.
- **Ne pas donner toutes les infos d'entrée** — laisser les agents chercher et comparer par eux-mêmes. Si on leur dit "Qwen 2.5 7B à $0.04/M est le moins cher", ils n'ont plus rien à évaluer. Donner des candidats, pas la réponse.
- **Spécifier les contraintes explicites** — "pas de local", "pas de gratuit", "via OpenRouter uniquement". Les agents ont tendance à proposer des solutions locales/Ollama si on ne leur dit pas d'éviter.
- **Vérifier les angles morts systématiquement** — les agents ont des biais : ils oublient facilement les modèles chinois (Qwen, DeepSeek hors V3/V4) et les modèles spécialisés. Un second tour est quasi toujours nécessaire.
- **Troncature des réponses** — le récap vitrine est limité à `MAX_CHARS` (5000 chars/réponse actuellement). Si les réponses sont coupées (`… (tronqué)`), voir `references/max-chars-truncation.md`.
- **Config du modèle auxiliaire post-turn** — après l'évaluation, la config à appliquer est `auxiliary.background_review` (et non "self_improvement"). Voir les détails et la décision finale dans `references/v018-background-review.md`.
- **Vérifier la fiabilité du modèle choisi** — un modèle parfait sur le papier peut faire des 503 (Gemini 2.5 Flash Lite via OpenRouter). Avant de trancher, Christophe peut demander aux agents de faire un test ping sur le candidat. Voir `references/v018-background-review.md` pour l'historique Gemini 503.

## Pièges

### Interception du slash par Element Matrix

**Double interception** possible :

1. **Côté Hermes** : les agents Matrix interprètent `/` comme une commande slash Hermes → `Unknown command /orch`. Solution : envoyer `orch status` (sans `/`), l'orchestrateur accepte les deux formes.

2. **Côté Element** (client Matrix) : Element intercepte tout message commençant par `/` comme une commande client. Il affiche « Commande non reconnue : /orch reset » et bloque l'envoi. Solutions :
   - Taper `//orch reset` (double slash) — le premier `/` échappe le second, Element envoie `/orch reset` en texte brut.
   - Taper simplement `orch reset` (sans slash) — plus simple, l'orchestrateur l'accepte aussi.

### Duplication Telegram → Matrix
Désactiver temporairement Matrix dans `config.yaml` pendant les travaux.

### Tokens Matrix
Token orchestrateur : `/opt/dendrite/data/orch_token.txt` sur le VPS.
⚠️ `/opt/dendrite/data/tokens.txt` (tokens centralisés) **n'existe plus** (constaté 04/07/2026) — les tokens utilisateurs sont dans la DB Dendrite (`userapi_accounts.db`) ; rotation = INSERT + `systemctl restart dendrite`.

### ⚠️ Obsolète en v4 : mute/unmute par PL supprimé
En v4, les agents ont un PL fixe 50 dans la vitrine. Le verrouillage est structurel (`events_default=51`, rooms privées). Tout le code de `set_power()`, `mute()`, `unmute()` de v3.2 a été supprimé.

### ⚠️ Obsolète en v4 : auto-correction PL supprimée
Le code d'auto-correction PL (lignes 169-175 de v3.2) n'existe plus dans l'orchestrateur v4. Plus besoin de forcer le PL de l'orchestrateur.

### Arrêt d'urgence (v4)
```bash
ssh root@79.143.180.202 "kill $(pgrep -f orchestrator_v4.py)"
```

### Approbations invisibles — agents orchestrateurs

Quand un agent orchestrateur (subagent de Chris) exécute une commande qui déclenche l'approval gate Hermes (`!approve`), personne ne peut l'approuver : les demandes ne sont visibles ni dans la vitrine Matrix, ni sur Telegram/Discord. L'agent reste bloqué indéfiniment.

**Solution appliquée** : le skill `kanban-worker` (chargé par les agents dispatchés) contient désormais une section « Do NOT » qui guide les agents à éviter les commandes déclenchant l'approval gate :
- Ne pas piper vers un interpréteur (`hermes | python3`)
- Préférer `execute_code` à `terminal` pour Python
- Utiliser les outils built-in (`read_file`, `search_files`, `patch`, `write_file`) plutôt que des commandes shell (`cat`, `grep`, `sed`, `echo`)

Si le problème persiste malgré le skill, la solution radicale est `delegation.subagent_auto_approve: true` dans `config.yaml` de Chris — mais cela désactive toute approbation pour les subagents, ce qui n'est pas souhaitable.

### Redémarrage orchestrateur v4

**Méthode recommandée — systemctl (service supervisé) :**
```bash
ssh root@79.143.180.202 "systemctl restart hermes-orchestrator-v4"
```
Le service est configuré avec `Restart=on-failure` et `RestartSec=10`. Avantage : une seule instance garantie, pas de doublon.

**Méthode manuelle (si systemd n'est pas dispo) — ATTENTION aux doublons :**
```bash
ssh root@79.143.180.202 "kill $(pgrep -f orchestrator_v4.py) 2>/dev/null; sleep 1"
ssh root@79.143.180.202 "cd /opt/dendrite && ORCH_TK=$(cat /opt/dendrite/data/orch_token.txt) && python3 -u orchestrator_v4.py >> data/orch_v4.log 2>&1 & disown"
```

**⚠️ Vérifier qu'une seule instance tourne après redémarrage :**
```bash
ssh root@79.143.180.202 "ps aux | grep orchestrator_v4 | grep -v grep"
```
Si deux processus apparaissent (PID distincts, même heure de lancement) → le `nohup ... &` a spawné un doublon, ou le service systemd tourne en parallèle. Symptôme : les messages partent en double (questions, récaps). Correction :
```bash
# Si service systemd actif → le laisser gérer, tuer l'instance manuelle
ssh root@79.143.180.202 "systemctl restart hermes-orchestrator-v4"
# Si pas de systemd → tuer le doublon
ssh root@79.143.180.202 "kill -9 <PID_doublon>"
```

**⚠️ Ne pas redémarrer via `systemctl` depuis une session Hermes** : `systemctl restart hermes-orchestrator-v4` est bloqué par le TIRITH guard d'Hermes (SIGTERM propagation). Solutions :
- Depuis un terminal SSH **extérieur** à la session Hermes.
- Depuis une session Hermes : utiliser `delegate_task` avec un subagent qui a un terminal frais.
- Si le service reste bloqué en `deactivating` après un SIGTERM, forcer :
  ```bash
  ssh root@79.143.180.202 "kill -9 $(systemctl show -p MainPID hermes-orchestrator-v4 --value 2>/dev/null)"
  ```
  systemd relance automatiquement (Restart=on-failure).

⚠️ **Attention** : ne pas laisser v3.2 et v4 tourner en même temps (conflit de sync). Vérifier : `ps aux | grep orchestrator`.

### Récap vitrine tronqué (« … (tronqué) » dans la vitrine)

Cause : `MAX_CHARS` dans `orchestrator_v4.py` (ligne 37) — limite de caractères par réponse dans la vitrine. Actuellement à 5000.

Quand Christophe signale des réponses tronquées :
1. Vérifier la valeur actuelle : `grep 'MAX_CHARS' /opt/dendrite/orchestrator_v4.py`
2. Augmenter si besoin (4000 couvre la quasi-totalité des cas)
3. Redémarrer le service systemd
4. Détail complet dans `references/max-chars-truncation.md`

### Timeout agents trop court — vérifier le BON timeout

Les constantes de timeout sont en haut de `/opt/dendrite/orchestrator_v4.py` :

```python
PAR_TIMEOUT = 240          # timeout global de collecte en mode parallèle
SEQ_TIMEOUT = 240          # timeout par agent en mode séquentiel
```

⚠️ **PIÈGE** : le mode par défaut est **parallèle**. Quand un agent met trop de temps, c'est `PAR_TIMEOUT` qu'il faut modifier, **pas** `SEQ_TIMEOUT`. Vérifier d'abord le mode actif avec `/orch status` avant de toucher aux constantes.

**Symptôme** : les agents répondent `(pas de réponse — timeout)` alors que l'agent a bien répondu en ~120-130s. L'orchestrateur a clôturé la collecte au bout de `PAR_TIMEOUT` secondes et le récap a été envoyé avant que la réponse n'arrive.

**Ajustement** :
```bash
ssh root@79.143.180.202 "sed -i 's/^PAR_TIMEOUT = [0-9]*/PAR_TIMEOUT = <nouvelle_valeur>/' /opt/dendrite/orchestrator_v4.py"
```
**Redémarrage obligatoire** : l'orchestrateur tourne comme un processus persistant (systemd), les modifications du fichier source ne prennent effet qu'après redémarrage.
```bash
ssh root@79.143.180.202 "pkill -9 -f orchestrator_v4.py; sleep 1; systemctl start hermes-orchestrator-v4"
```

### Changement de sujet — forcer un `/orch reset`

Quand Christophe change complètement de sujet, les 5 derniers échanges (injectés dans le contexte des agents via `cycle_history`) peuvent parasiter l'analyse. Les agents reçoivent le nouveau message préfixé de questions/réponses hors-sujet → confusion, réponses à côté.

**Solution** : `/orch reset` vide `cycle_history` ET envoie `@agent /new` dans chaque room privée (réinitialise aussi les sessions agent). Les agents repartent avec un contexte vierge, sans historique parasite.

### Logs muets mais state actif
Le state file (`/opt/dendrite/data/orchestrator_state_v4.json`) est la source de vérité. `cycle_status: "idle"` + `current_question` défini = cycle avorté → Christophe renvoie son message.
⚠️ **Chemin exact** : le fichier est dans `/opt/dendrite/data/` **pas** à la racine `/opt/dendrite/`.

### Vérifier que le récap a bien été diffusé
Zéro erreur HTTP dans le log = les envois réussissent. Vérifier :
```bash
# Compter les erreurs HTTP
grep -c 'HTTP\|REQ ERROR' /opt/dendrite/data/orch_v4.log
# Voir les fins de cycle
grep 'CYCLE END\|reca\|Récap\|diffus' /opt/dendrite/data/orch_v4.log | tail -10
# Lire le log complet
cat /opt/dendrite/data/orch_v4.log
```

### Threads Matrix — contournement du verrouillage PL
Les agents ont PL 50 dans la vitrine, `events_default=51` → ne peuvent pas poster dans la room principale. **Mais dans les threads Matrix**, le verrouillage PL peut être contourné selon la configuration Dendrite :
- Un message en réponse à un thread existant n'est pas toujours soumis aux mêmes `events_default`
- Résultat : des agents peuvent apparaître dans la vitrine en répondant à un thread de Christophe
- **Solution** : vérifier régulièrement les PL du thread ou configurer Dendrite pour appliquer les PL aux threads aussi. Commande de diagnostic :
  ```bash
  curl -s -H "Authorization: Bearer $(cat /opt/dendrite/data/orch_token.txt)" \
    "https://matrix.videocours.fr/_matrix/client/v3/rooms/!McOijzMcBSlvlfId:videocours.fr/state/m.room.power_levels"
  ```
Détail des commandes de diagnostic et interprétation des logs : `references/orchestrator-debugging.md`.

### Pairing Matrix — utilisateur non approuvé (« Unauthorized user »)

Si un agent ignore les messages de l'orchestrateur avec `Unauthorized user: @orchestrateur:... on matrix`,
le fichier de pairing Matrix n'existe pas ou ne contient pas l'orchestrateur.

**Solution** : créer le fichier `matrix-approved.json` dans le dossier de pairing Hermes.
⚠️ **Piège** : `get_hermes_dir("platforms/pairing", "pairing")` utilise l'ANCIEN chemin
(`$HERMES_HOME/pairing/`) si ce dossier existe déjà. Ne pas confondre avec le NOUVEAU chemin
(`$HERMES_HOME/platforms/pairing/`). Vérifier via Python :

```python
from hermes_constants import get_hermes_dir
print(get_hermes_dir("platforms/pairing", "pairing"))  # chemin réel
```

Pour un agent avec `HERMES_HOME=/opt/data` où `/opt/data/pairing/` existe déjà, le fichier
doit être dans `/opt/data/pairing/matrix-approved.json`, PAS dans `/opt/data/platforms/pairing/`.

```json
{"@orchestrateur:videocours.fr": {"user_name": "orchestrateur", "approved_at": 1719600000.0}}
```

Puis redémarrer le gateway. Voir `references/matrix-pairing.md` pour le détail complet.

### Systemd — orchestrateur v4 supervisé

```bash
# /etc/systemd/system/hermes-orchestrator-v4.service
[Unit]
Description=Hermes Orchestrateur Matrix v4
After=network.target dendrite.service
Wants=dendrite.service

[Service]
Type=simple
User=root
WorkingDirectory=/opt/dendrite
Environment=ORCH_TK=<token>           # coller le contenu de orch_token.txt
ExecStart=/usr/bin/python3 -u orchestrator_v4.py
Restart=on-failure
RestartSec=10
StandardOutput=append:/opt/dendrite/data/orch_v4.log
StandardError=append:/opt/dendrite/data/orch_v4.log

[Install]
WantedBy=multi-user.target
```

**Installation :**
```bash
# 1. Récupérer le token
TOK=$(cat /opt/dendrite/data/orch_token.txt)

# 2. Écrire le service (avec le token en dur dans Environment)
cat > /etc/systemd/system/hermes-orchestrator-v4.service <<SVC
...
Environment=ORCH_TK=$TOK
...
SVC

# 3. Activer + démarrer
systemctl daemon-reload
systemctl enable hermes-orchestrator-v4
systemctl start hermes-orchestrator-v4
systemctl status hermes-orchestrator-v4   # vérifier active (running)
```

**Piège EnvironmentFile** : systemd lit `EnvironmentFile=/path` mais le format attendu est `KEY=VALUE` sur chaque ligne. Si le fichier contient `ORCH_TK=$(cat ...)` (littéral, non expansé), la variable vaut la chaîne `$(cat ...)` → M_UNKNOWN_TOKEN. Solution : soit mettre le token en dur dans `Environment=ORCH_TK=<valeur>`, soit s'assurer que le fichier contient bien le token réel.

### Verrouillage vitrine (events_default=51)

Appliqué le 05/07/2026 : `PUT rooms/!McOijz.../state/m.room.power_levels` avec `events_default=51`.
Les agents ont PL 0, Christophe 100, orchestrateur 51. Aucun agent ne peut poster dans hermes-team,
même avec un PL supérieur — `events_default=51` bloque tout message non-admin.
Testé et confirmé : `PUT OK` via l'API Matrix.

### Délai orchestration → agent : ~4-30 s

Entre l'envoi du récap par l'orchestrateur et sa réception par l'agent :

| Étape | Durée |
|---|---|
| API PUT (orchestrateur → Dendrite) | ~0.5-1 s |
| Stockage Dendrite + publication event | < 0.1 s |
| ⏳ Attente sync gateway agent (timeout 5000 ms) | ~3-5 s |
| Parsing + dispatch gateway | ~1-2 s |
| ⏳ Inférence LLM (si réponse) | ~5-15 s |

**Au mieux** (sync tombe au bon moment) : ~4-6 s. **Au pire** (juste après un sync) : ~10-12 s.
**Avec réponse LLM** : ~15-30 s au total.

### Récap dans les rooms privées : la question est incluse

Le récap envoyé dans chaque room privée contient la question de Christophe + les 4 réponses :
```
@agent:videocours.fr 📋 Récap (informatif — NE PAS RÉPONDRE)

📋 Récap — Question : <texte de la question>

[SOS] ...
[THREE] ...
[FIVE] ...
[CHRIS] ...
```

Le code utilise `nl.join(lines_full)` (pas `[1:]`) pour que la question soit dans le récap.
⚠️ Un bug initial utilisait `[1:]` qui sautait la ligne de la question — corrigé en cours de migration.

### Redémarrage gateway Chris (VPS Hostinger Docker)
Chris tourne dans un conteneur Docker `hvps-hermes-agent` sur `187.124.223.135`. Pas de systemd à l'intérieur :
```bash
ssh root@187.124.223.135
docker exec hermes-agent-jjqz-hermes-agent-1 \
  nohup hermes gateway run >> /opt/data/gateway.log 2>&1 &
```
Le gateway redémarre, vérifier avec `hermes gateway status`.

### is_system_msg() peut filtrer une réponse légitime
Le filtre hérité de v3.2 bloque les messages contenant `📋`, `todo:`, `📬`, etc.
Si un agent utilise ces patterns dans une réponse normale, il est ignoré et
compte comme n'ayant pas répondu. Détail dans `references/orchestrator-v4-migration.md`.

### Capture de réponse agent dans l'orchestrateur — syndrome du « bruit avant réponse »

**Problème** : quand un agent Hermes reçoit un message Matrix, son gateway envoie souvent plusieurs messages avant la réponse réelle :
- `📬 No home channel is set for Matrix.` (warning gateway)
- `📚 Reading skill <nom>` (chargement de skill)
- `⚠️ Empty response from model — retrying (1/3)` (retry LLM)

Ces messages arrivent AVANT la réponse utile, et si l'orchestrateur capture le premier message sans filtrer, il consomme le flag de capture sur du bruit.

**Pattern correct** : buffer + filtre + finalisation différée.

```python
NOISE_PATTERNS = [
    'No home channel is set for Matrix',
    'Reading skill',
    'Empty response from model',
    'Model returned no content',
    'No reply: the model returned empty',
]

# Phase 1 — Buffériser (filtrer le bruit, accumuler les vrais messages)
if st.get("synth_waiting") and agent == "@chris:videocours.fr":
    if any(p in body for p in NOISE_PATTERNS):
        log("bruit ignore: " + str(len(body)) + " chars")
        continue
    buf = st.get("synth_buffer", [])
    buf.append(body)
    st["synth_buffer"] = buf
    st["synth_last_msg"] = time.time()
    continue

# Phase 2 — Finaliser après silence (dans la boucle principale)
if st.get("synth_waiting") and st.get("synth_last_msg") and time.time() - st["synth_last_msg"] > 8:
    buf = st.get("synth_buffer", [])
    combined = chr(10).join(buf)
    synth_msg = "🧠 Synthese :" + chr(10) + combined[:3000]
    send(synth_msg)                                    # vitrine
    for a in PRIVATE_ROOMS:
        send_to(PRIVATE_ROOMS[a], synth_msg)           # chaque room privée
    st["synth_waiting"] = False
    st.pop("synth_buffer", None)
    st.pop("synth_last_msg", None)
```

Ce pattern est utilisé pour `orch synthese` mais est transposable à toute capture de réponse agent dans l'orchestrateur v4.

**⚠️ Piège supplémentaire** : les scripts de nettoyage automatique (indentation fixer, linter) peuvent commenter des blocs qui semblent sous-indentés. Vérifier après chaque modification que le bloc de capture est actif et non commenté. La commande `grep -n 'Capture reponse\|synth_waiting' orchestrator_v4.py` doit montrer des lignes SANS `#` en début de bloc.