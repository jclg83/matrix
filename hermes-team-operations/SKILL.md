---
name: hermes-team-operations
description: Règles de fonctionnement de l'équipe Hermes — tours de parole, anti-boucle, partage de skills, décisions.
version: 1.7.0
metadata:
  hermes:
    tags: [team, orchestration, rules, orchestrator-v3, commands]
---

# Hermes Team Operations

## Règle de parole
Ordre nominal : SOS → Three → Five → Chris.
Un agent ne répond que si :
- Christophe le sollicite directement (@agent)
- L'orchestrateur lui donne explicitement la parole
- Un autre agent lui passe la main nommément

**Jamais** répondre à un message qui ne t'est pas adressé.

**Power levels** : Dendrite enforce bien les power levels Matrix (testé et vérifié le 29/06/2026 — un utilisateur avec power level 0 est bien bloqué par `M_FORBIDDEN`).

⚠️ **Règle Matrix critique** : un utilisateur de niveau N ne peut **pas** modifier un autre utilisateur de niveau ≥ N. Donc un orchestrateur à 50 **ne peut pas muter** un agent à 50 (50 ≥ 50 → `M_FORBIDDEN`). D'où le design (appliqué le 30/06/2026) :

| Niveau | Qui | Effet |
|---|---|---|
| 100 | Christophe | Admin, contrôle total |
| **51** | Orchestrateur | Modérateur (peut mute 50 car 51 > 50) |
| **50** | Agents (démutés) | Peut parler (`events_default` = 50) |
| 0 | Agents (mutés) | Bloqué |

`events_default` pour `m.room.message` = **50** (inchangé). Seul Christophe (PL 100) peut modifier les power levels — l'orchestrateur n'a pas ce droit, même à 51. Pour appliquer la config, utiliser le token Christophe (trouvable dans `/opt/dendrite/data/tokens.txt`).

⚠️ **Piège auto-correction PL** : au démarrage, l'orchestrateur force son propre PL à **51** (ligne ~171 de `orchestrator.py`). Le code a été corrigé le 30/06/2026 pour refléter le nouveau design.

Le mute/démute de l'orchestrateur a un **effet réel côté serveur**. L'anti-boucle v3 (ignorer les messages hors tour) reste une sécurité supplémentaire.

## Commandes orchestrateur (`/orch` ou `orch`)

Christophe contrôle l'orchestrateur via 15 commandes dans la room Matrix.
**Formes acceptées** : `/orch <commande>` (slash Matrix) et `orch <commande>` (message normal, sans slash). 
Utiliser `orch` sans slash évite l'interception par les bots Hermes Matrix.

| Commande | Effet |
|---|---|
| `/orch status` | Cycle, speaker, ordre, file d'attente |
| `/orch stop` | Arrête le cycle en cours |
| `/orch pause` | Suspend les nouveaux cycles |
| `/orch resume` | Réactive l'orchestrateur |
| `/orch skip` | Passe au speaker suivant |
| `/orch ordre A,B,C,D` | Change l'ordre de parole |
| `/orch default-order` | Restaure l'ordre par défaut |
| `/orch agents` | Liste les agents et leur état |
| `/orch disable @agent` | Retire un agent des cycles |
| `/orch enable @agent` | Réintègre un agent |
| `/orch queue` | Affiche les questions en attente |
| `/orch clearqueue` | Vide la file d'attente |
| `/orch reset` | Reset complet du state |
| `/orch log [N]` | N dernières lignes de log |
| `/orch muteall` | Mute tous les agents immédiatement |
| `/orch unmuteall` | Réactive tous les agents mutés |
| `/orch help` | Liste toutes les commandes |

**Important** : `disable`/`enable` retire/réintègre l'agent du cycle côté orchestrateur. `muteall`/`unmuteall` est un raccourci pour disabler tous les agents d'un coup, persistant via le state file. Le mute Matrix (power level 0 → bloqué par Dendrite) reste le mécanisme de base ; `disable` est un cran supplémentaire.

## Séquenceur visible (v3.3)

L'orchestrateur édite un message de statut en direct dans la room :

| Événement | Message |
|---|---|
| Début de cycle | `🔄 Cycle | Tour : SOS | Ordre : SOS → Three → Five → Chris | Question : ...` |
| Changement de tour | Le même message est **édité** (pas de nouveau message) |
| Fin normale | `✅ Cycle terminé` |
| Timeout | `⏰ Cycle terminé (timeout)` |
| `/orch stop` | `🛑 Cycle arrêté par /orch stop` |

Le message s'auto-édite via `m.replace` — pas de spam, un seul message qui évolue.

## Configuration Matrix pour les agents

Chaque agent doit avoir sa config Matrix correctement paramétrée :

```yaml
matrix:
  enabled: true
  user_id: "@agent:videocours.fr"
  server_url: "https://matrix.videocours.fr"
  access_token: "<token depuis /opt/dendrite/data/tokens.txt>"
  allowed_rooms: "!McOijzMcBSlvlfId:videocours.fr"
  require_mention: false
```

**⚠️ `require_mention: false` obligatoire** pour que l'agent réponde quand l'orchestrateur lui donne la parole. L'orchestrateur mentionne l'agent (`@agent:videocours.fr`), mais `require_mention: false` garantit que l'agent traite le message même sans être le seul destinataire.

**⚠️ `interrupt_notifications: false` obligatoire** — les notifications d'interruption (`⚡️ Interrupting current task`, `Operation interrupted`) polluent la room Matrix et déclenchent les autres agents. Désactiver avec :

```yaml
platforms:
  telegram:
    streaming:
      interrupt_notifications: false  # déjà fait
  matrix:
    streaming:
      interrupt_notifications: false  # ← À AJOUTER
```

Si la config Matrix n'a pas de section `streaming`, l'ajouter manuellement.

## Mise à jour du skill

Le skill est versionné sur GitHub : `jclg83/matrix` → `hermes-team-operations/SKILL.md`

Pour mettre à jour votre copie locale :
```bash
mkdir -p ~/AppData/Local/hermes-sos/skills/team/hermes-team-operations
curl -o ~/AppData/Local/hermes-sos/skills/team/hermes-team-operations/SKILL.md   https://raw.githubusercontent.com/jclg83/matrix/main/hermes-team-operations/SKILL.md
```
Puis redémarrer Hermes.

## Anti-boucle

### Règles côté agent (OBLIGATOIRES)

**Filtrage Matrix — l'agent DOIT ignorer :**

1. **Messages des autres agents** — ne JAMAIS répondre à `@sos`, `@three`, `@five`, `@chris` sauf si :
   - Le message vient de l'orchestrateur ET te mentionne explicitement (ex: `@five:videocours.fr <question>`)
   - Christophe te mentionne directement (`@five`)

2. **Messages système Hermes** — ignorer tout message contenant :
   - `⚡️ Interrupting current task`
   - `Operation interrupted`
   - `Interrupt recursion depth`
   - `📬 No home channel`
   - `Model returned no content`
   - `Unknown command /orch`
   - `Silence.` (envoyé par un autre agent)

3. **Réactions Matrix** — ne pas réagir aux emojis/réactions des autres agents

**En pratique :** quand un message arrive sur Matrix, vérifier AVANT de répondre :
- `sender == @christophe:videocours.fr` → répondre (question ou mention)
- `sender == @orchestrateur:videocours.fr` ET `@mon_nom:videocours.fr` dans le body → répondre (tour de parole)
- Tout le reste → **IGNORER, ne pas répondre, ne pas accuser réception**

### Règles générales
- Ne pas envoyer d'accusés de réception (« reçu », « j'attends », « ok »)
- Ne pas commenter les messages des autres agents sauf si sollicité
- Une réponse = un seul message. Pas de suivi automatique.
- En cas de doute, attendre une consigne explicite.

### Côté orchestrateur (technique)
- **v3** : l'orchestrateur ignore tout message d'un agent qui n'est pas `current_speaker`.
- **Filtre `is_system_msg()`** : l'orchestrateur filtre les patterns système (`Interrupting current task`, `📬 No home channel`, etc.).

## Tâches longues
Le tour de parole ne bloque que les messages dans la room.
Un agent peut travailler en arrière-plan si Christophe lui a confié une mission.

## Skills partagés
Les skills communes sont versionnées dans le dépôt Git `jclg83/matrix` (GitHub).
Chaque agent les installe depuis cette source unique.

## Décision humaine
Christophe est le seul décideur. Les agents proposent, Christophe tranche.
Ne jamais déployer, modifier une config, ou lancer une action sans GO explicite.

## Pièges

### `unmute()` utilisait 49 → bloqué par `events_default` 50
**Corrigé le 30/06/2026** : `set_power(agent, 49)` → `set_power(agent, 50)`.
Vérification : `grep -n "set_power(agent" /opt/dendrite/orchestrator.py`

### Auto-correction PL forçait 50 au lieu de 51
**Corrigé le 30/06/2026** : `current_lvl != 50` → `current_lvl != 51`.

### Interception du slash par les agents Hermes
Envoyer `orch status` (sans `/`) au lieu de `/orch status`.

### Duplication Telegram → Matrix
Désactiver temporairement Matrix dans `config.yaml` pendant les travaux.

### Tokens Matrix
Stockés dans `/opt/dendrite/data/tokens.txt` sur le VPS.

### Arrêt d'urgence
```bash
ssh root@79.143.180.202 "kill $(pgrep -f orchestrator.py)"
```

### Redémarrage orchestrateur
```bash
ssh root@79.143.180.202 "kill $(pgrep -f orchestrator.py) 2>/dev/null; sleep 1"
ssh root@79.143.180.202 "cd /opt/dendrite && ORCH_TK=$(cat /opt/dendrite/data/orch_token.txt) nohup python3 -u orchestrator.py >> orchestrator.log 2>&1 &"
```

### Logs muets mais state actif
Le state file (`orchestrator_state.json`) est la source de vérité. `cycle_status: "idle"` + `current_question` défini = cycle avorté → Christophe renvoie son message.
