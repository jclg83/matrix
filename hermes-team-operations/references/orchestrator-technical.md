# Orchestrateur Matrix — v3.2 (état de cycle + 15 commandes /orch)

## Où ça tourne

- **Serveur** : VPS videocours.fr (`79.143.180.202`)
- **Script** : `/opt/dendrite/orchestrator.py` (~15500 chars) + module `/opt/dendrite/orch_commands.py` (~4300 chars)
- **Backup** : `/opt/dendrite/orchestrator.py.bak` (v2)
- **Lancement** : `cd /opt/dendrite && ORCH_TK=$(cat /opt/dendrite/data/orch_token.txt) nohup python3 -u orchestrator.py >> /opt/dendrite/data/orch_v2.log 2>&1 &`
- **State** : `/opt/dendrite/data/orchestrator_state.json`
- **Token** : `/opt/dendrite/data/orch_token.txt`
- **Copie locale (debug)** : `C:\Users\strap\hermes-sos-workspace\orchestrator_v3.py` (sur le PC portable 4090, version figée utilisée pour analyse hors VPS). Accessible en Python via `/mnt/c/Users/strap/hermes-sos-workspace/orchestrator_v3.py` (MSYS).

## Paramètres

| Paramètre | Valeur | Rôle |
|---|---|---|
| `HOMESERVER` | `https://matrix.videocours.fr` | Serveur Dendrite |
| `ROOM_ID` | `!McOijzMcBSlvlfId:videocours.fr` | Room `hermes-team` |
| `USER_ID` | `@orchestrateur:videocours.fr` | Compte Matrix de l'orchestrateur |
| `ADMIN_USER` | `@christophe:videocours.fr` | Seul déclencheur de cycles |
| `ALL_AGENTS` | `@sos, @three, @five, @chris` | Agents reconnus |
| `DEFAULT_ORDER` | SOS → Three → Five → Chris | Ordre de parole par défaut |
| `MAX_CHARS` | `1000` | Défini mais pas encore appliqué |
| `TIMEOUT_SEC` | `90` | Timeout si un agent ne répond pas |
| `POLL_INT` | `3` | Intervalle de polling (secondes) |

## Fonctionnement v3

1. Long-polling Matrix `/sync` (timeout 5s) sur la room
2. Christophe écrit → orchestrateur détecte (sauf si @mention d'un agent)
3. Remplit l'état de cycle : `cycle_id`, `current_speaker`, `allowed_speakers`, `cycle_status = "running"`
4. Démute le 1er agent, lui envoie la question
5. Agent répond → orchestrateur le mute, met à jour `current_speaker`, passe au suivant
6. **Anti-boucle** : tout message d'un agent qui n'est PAS `current_speaker` → ignoré + log
7. Timeout de 90s par agent → mute forcé + passage au suivant
8. Fin de cycle : `cycle_status = "idle"`, reset `current_speaker`/`allowed_speakers`/`cycle_id`
9. Questions en attente (pendant un cycle) → stockées dans `st["pending_questions"]` persistant

## Nouveautés v3 vs v2

| Fonctionnalité | v2 | v3 |
|---|---|---|
| État de cycle | Aucun | `cycle_id`, `current_speaker`, `cycle_status`, `allowed_speakers` |
| Anti-boucle | ❌ | ✅ Message hors tour → ignoré |
| Reprise après crash | ❌ Cycle perdu | ✅ Annule le cycle, notifie Christophe |
| Pending questions | Variable locale `pending` | `st["pending_questions"]` persistant |
| Save atomique | ❌ `open().write()` | ✅ `.tmp` + `os.replace()` |
| Commande `ordre:` | Simple | + confirmation `✅ Ordre mis à jour : SOS → Three → ...` |

## State v3 (`orchestrator_state.json`)

```json
{
  "order": ["@sos:...", "@three:...", "@five:...", "@chris:..."],
  "last_sync_token": "s8406_...",
  "seen_events": ["$OgkpTr...", ...],
  "cycle_id": "cycle_1719600000",
  "current_speaker": "@three:videocours.fr",
  "allowed_speakers": ["@three:videocours.fr"],
  "cycle_status": "running",
  "current_question": "quelle est la météo ?",
  "pending_questions": [
    {"question": "et demain ?", "admin_event_id": "$abc123", "created_at": 1719600123.45}
  ],
  "last_admin_event_id": "$xyz789",
  "disabled_agents": ["@three:videocours.fr", "@five:videocours.fr"]
}
```

## Commandes `/orch` (v3.2 — implémenté)

Toutes les commandes doivent être envoyées par Christophe (`ADMIN_USER`) dans la room.

### Contrôle du cycle

| Commande | Effet |
|---|---|
| `/orch status` | Cycle, speaker, ordre, file d'attente |
| `/orch stop` | Arrête le cycle en cours, mute tous les agents |
| `/orch pause` | Suspend les nouveaux cycles (ne coupe pas le cycle en cours) |
| `/orch resume` | Réactive l'orchestrateur |
| `/orch skip` | Passe au speaker suivant immédiatement |

### Gestion de l'ordre

| Commande | Effet |
|---|---|
| `/orch ordre A,B,C,D` | Change l'ordre de parole |
| `/orch default-order` | Restaure SOS → Three → Five → Chris |

### Gestion des agents

| Commande | Effet |
|---|---|
| `/orch agents` | Liste tous les agents et leur état (SPEAKER/enabled/disabled) |
| `/orch disable @agent` | Retire un agent des cycles (l'orchestrateur l'ignore) |
| `/orch enable @agent` | Réintègre un agent |
| `/orch muteall` | **v3.2+** — Mute tous les agents d'un coup (ajoute à `disabled_agents`, persistant) |
| `/orch unmuteall` | **v3.2+** — Réactive tous les agents mutés |

### File d'attente

| Commande | Effet |
|---|---|
| `/orch queue` | Affiche les questions en attente (avec horodatage) |
| `/orch clearqueue` | Vide la file d'attente |

## Limites connues v3.2

| Limite | Impact | Statut |
|---|---|---|
| `MAX_CHARS` pas appliqué | Défini (`1000`) mais non utilisé dans le code | P2 |
| Pas de systemd | Crash → pas de redémarrage automatique | P0 (à faire) |
| Pas de limite de temps de parole | Un agent peut monopoliser (sauf timeout 90s) | ⚠️ |
| `/orch` implémenté ✅ | 15 commandes fonctionnelles via module `orch_commands.py` | ✅ |
| **Power levels enforceables** ✅ | Dendrite bloque bien — testé 29/06 (`M_FORBIDDEN 0 < 50`). Mute/démute effectif. | ✅ |
| **Logging mute/démute** ✅ | `set_power()` logge chaque opération : `DEMUTE sos: 0 → 50 [OK]` | ✅ |
| **Auto-correction niveau** ⚠️ | Au démarrage, vérifie `@orchestrateur = 50`, corrige si besoin — **BUG avec PL=51** : force à 50 alors qu'on veut 51. À corriger dans le code : `if current_lvl != 51` et `pl[\"users\"][USER_ID] = 51`. | ⚠️ |
| **`unmute()` hardcode 49** ❌ | `def unmute(agent): return set_power(agent, 49)` — or `events_default` = 50 donc l'agent ne peut pas poster même « démuté ». Doit être `set_power(agent, 50)`. Bug silencieux (log OK mais agent muet). | ❌ Corrigé 30/06 |
| **Tokens centralisés** ℹ | `/opt/dendrite/data/tokens.txt` contient tous les tokens utilisateurs (christophe, sos, five, three, chris) en clair. | ℹ |
| **Filtre temporel 60s** ⚠️ | Messages >60s ignorés → `/orch` pendant downtime perdus | ⚠️ Design |
| **Pas de `emergency_stop.py`** ❌ | Le script `/opt/dendrite/scripts/emergency_stop.py` n'existe pas en production | P1 |

## Filtre `is_system_msg()` (v3)

L'orchestrateur ignore les messages système pour éviter les faux déclenchements. Patterns filtrés :

**Contenu textuel** (insensible à la casse) :
- `No home channel is set`
- `Operation interrupted`
- `Interrupting current task`
- `skill_view`, `session_search`, `search_files`
- `Codex gpt-5.5 caps`
- `Self-improvement review`
- `cronjob:`, `todo:`

**Préfixes emoji** : 📬, 📚, 🔍, 🔎, 💻, ⏰, 📋, 💾, ⚡, ℹ

**Non filtré (limitation connue)** : `⚠` (warning), `⚠️ The model returned no response`, messages préfixés par `* ` (ex: `* 🔍 session_search` — mais le pattern textuel `session_search` les rattrape)

Ce filtre s'applique APRES le check anti-boucle (agent hors-tour), donc un message système d'un agent hors-tour est ignoré deux fois. Tout message filtré est loggé (`System msg from @agent - ignored`) mais ne déclenche PAS l'avancement du cycle.

**Effet cascade d'interruptions** : quand l'orchestrateur envoie un message (ping, status, sollicitation), le plugin Matrix de chaque Hermes Agent interrompt l'agent en cours (`⚡ Interrupting current task`). Chaque agent répond par un acquittement… que l'orchestrateur filtre via `is_system_msg()`. Résultat : la room est bruyante mais le cycle reste stable.

**Mitigation côté client** : chaque agent doit avoir `matrix.streaming.interrupt_notifications: false` dans son `config.yaml` pour supprimer ces messages à la source (Hermes n'émet plus les `⚡ Interrupting current task` dans la room). Voir `agent-matrix-config.md`.

## Redémarrage

⚠️ L'orchestrateur lit le token depuis `ORCH_TK` (variable d'env), pas depuis le fichier.

### Procédure complète avec nettoyage

```bash
# 1. Tuer tout processus orchestrateur existant
ssh root@79.143.180.202 "kill \$(pgrep -f orchestrator.py) 2>/dev/null; sleep 1"

# 2. Nettoyer le lock stale
ssh root@79.143.180.202 "rm -f /opt/dendrite/data/orchestrator.lock"

# 3. Lancer proprement (cd dans /opt/dendrite pour les imports relatifs)
ssh root@79.143.180.202 "cd /opt/dendrite && ORCH_TK=\$(cat /opt/dendrite/data/orch_token.txt) nohup python3 -u orchestrator.py >> /opt/dendrite/data/orch_v2.log 2>&1 &"

# 4. Vérifier UN SEUL processus (pas de doublon)
ssh root@79.143.180.202 "ps aux | grep orchestrator | grep -v grep"
```

### Piège : processus en double

Si `pkill` rate une instance zombie (ex: processus lancé avec `python3 -u /opt/dendrite/orchestrator.py` vs `python3 -u orchestrator.py`), le `pgrep -f orchestrator.py` peut rater une des deux formes. **Solution** : vérifier `ps aux | grep orchestrator` après lancement — un seul processus doit apparaître. Si deux processus tournent → cycles en conflit, comportement imprévisible.

### Piège : phase silencieuse après restart (catch-up)

Après un redémarrage avec un token de sync ancien, l'orchestrateur peut rester **plusieurs minutes sans aucune entrée de log**. Ce n'est PAS un crash — il est en phase de catch-up :
- Chaque sync ramène 50 événements (max 50 par `timeline.limit`)
- Tous les événements sont ajoutés à `seen` mais filtrés par le filtre 60s → aucun `MSG` loggé
- La sauvegarde d'état ne se déclenche que quand `len(seen) % 20 == 0`
- Cycle typique : 5s (sync timeout) + ~0s (processing) + 3s (POLL_INT) = ~8s par batch de 50

**Comment diagnostiquer** : vérifier si le token de sync dans le state file avance :
```bash
ssh root@79.143.180.202 "python3 -c \"
import json, time
for i in range(3):
    with open('/opt/dendrite/data/orchestrator_state.json') as f:
        tok = json.load(f).get('last_sync_token','')[:40]
    print(f't{i}: {tok}')
    time.sleep(5)
\""
```
Si le token change → l'orchestrateur fonctionne, il rattrape juste son retard. Attendre que le token se stabilise (rattrapage terminé), puis Christophe peut envoyer `/orch`.

### Piège : code d'initialisation hors try/except

L'auto-correction des power levels (lignes 169-175) s'exécute **avant** la boucle principale `while True` et donc **en dehors** du `try/except Exception` qui protège la boucle. Si `req("GET", "rooms/.../power_levels")` lève une exception non-HTTP (ex: `KeyError` sur `pl["users"]` si la réponse est malformée), le processus crash **sans aucune entrée dans le log**. Le processus disparaît de `ps aux` sans trace.

### Piège : filtre temporel 60 secondes

L'orchestrateur ignore les événements Matrix dont `origin_server_ts` est plus ancien que 60 secondes (ligne ~193 de `orchestrator.py`) :
```python
if body and sender != USER_ID and (time.time()*1000 - ev.get("origin_server_ts",0)) < 60000:
```

**Conséquence après un crash/restart** : tous les messages (y compris les `/orch`) émis pendant le downtime sont filtrés car leur timestamp est >60s. Une fois l'orchestrateur redémarré, Christophe doit **renvoyer** sa commande `/orch` — elle sera capturée immédiatement puisque l'orchestrateur écoute en temps réel.

### Vérification post-redémarrage

```bash
# L'orchestrateur tourne ?
ssh root@79.143.180.202 "ps aux | grep orchestrator | grep -v grep"

# Dernières entrées du log (doit montrer des MSG récents si la room est active)
ssh root@79.143.180.202 "tail -10 /opt/dendrite/data/orch_v2.log"

# État du cycle
ssh root@79.143.180.202 "python3 -c \"
import json
with open('/opt/dendrite/data/orchestrator_state.json') as f:
    s = json.load(f)
for k in ('cycle_status','current_speaker','current_question','cycle_id'):
    print(f'{k}: {s.get(k)}')
\""
```

## Déploiement d'une mise à jour

⚠️ **Toujours backup d'abord** : `cp /opt/dendrite/orchestrator.py /opt/dendrite/orchestrator.py.bak`

### Méthode 1 : Script de patch Python
Écrire un patch sur le VPS via SSH, l'exécuter, redémarrer. Voir `matrix-dendrite-videocours/references/orchestrator-v3-upgrade.md`.

### Méthode 2 : Base64 (quand les caractères spéciaux sont corrompus)
Pour contourner le bug #15236 (corruption de `(`, `)`, `"` dans les commandes SSH) :
```bash
# Local : encode le script Python en base64
# VPS : écrit en 2 echo, décode, exécute
ssh root@VPS 'echo AAA...base64_part1... > /tmp/script.b64'
ssh root@VPS 'echo BBB...base64_part2... >> /tmp/script.b64'
ssh root@VPS 'base64 -d /tmp/script.b64 | python3'
```

### Pièges
- **Redaction Hermes des tokens** : la ligne `ACCESS_TOKEN=open("/path").read().strip()` est systématiquement tronquée. Le script en production lit le token depuis `/opt/dendrite/data/orch_token.txt` au runtime.
- **SSH heredoc + Python `f.read()`** : le pattern `f.read().strip()` dans un heredoc SSH (`<< 'EOF'`) est mal interprété — `f"""` est vu comme le début d'une triple-quoted f-string, cassant la syntaxe Python. **Workaround** : utiliser un nom de variable différent de `f` (ex: `fh = open(path); TOK = fh.read().strip(); fh.close()`) ou encoder le script en base64 avant de l'envoyer.
- **Duplication Matrix** : désactiver `matrix: enabled: false` dans `config.yaml` + `/restart` avant de travailler sur l'orchestrateur.
