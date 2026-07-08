# Migration v3 → v4 — Orchestrateur Matrix

**Migré le 04/07/2026** par SOS, sous les ordres de Christophe.
v4 tourne depuis le 05/07/2026 via systemd.

## Résumé des changements

| Avant (v3.2) | Après (v4) |
|---|---|
| 1 room unique (hermes-team) | 1 vitrine + 4 rooms privées |
| mute/unmute PL dynamique (`set_power`) | Zéro PL dynamique — verrouillage structurel |
| Tour de parole séquentiel | Collecte parallèle par défaut |
| `current_speaker`, `allowed_speakers` dans le state | Supprimés — remplacés par `responses` dict |
| Pas de contexte cyclique | `cycle_history` (5 derniers cycles) dans chaque question |
| Récap sans @mention | `@agent` + question incluse dans le récap |
| Pas de service → crash = arrêt | systemd avec `Restart=on-failure` |

## Étapes de migration (ordre réel)

### 1. Création des 4 rooms privées
Sur le VPS Dendrite via `createRoom` API :
```bash
curl -X POST -H "Authorization: Bearer $ORCH_TK" \
  https://matrix.videocours.fr/_matrix/client/v3/createRoom \
  -d '{"preset": "private_chat", "name": "orch-sos", "invite": ["@sos:videocours.fr", "@christophe:videocours.fr"]}'
```
Mapping sauvegardé dans `/opt/dendrite/data/private_rooms.json`.

### 2. Patch @mention dans question_text
v3.2 envoyait la question sans @mention. Les agents avec `require_mention: true` ignoraient le message.
**Correction** : `question_text(agent, q)` → `question_text(agent, q, st)` avec `@agent:... ❓ ...`.

### 3. Bascule des agents
Chaque agent change `matrix.allowed_rooms` dans sa config → uniquement sa room privée.
Redémarrage du gateway obligatoire.

### 4. Pairing Matrix (cas Chris)
Problème : `Unauthorized user: @orchestrateur:videocours.fr on matrix`.
**Cause** : fichier `matrix-approved.json` manquant dans le dossier de pairing.
**Piège** : `get_hermes_dir("platforms/pairing", "pairing")` utilise l'ANCIEN chemin
`$HERMES_HOME/pairing/` si ce dossier existe déjà (et non `platforms/pairing/`).
**Solution** : créer `$HERMES_HOME/pairing/matrix-approved.json` avec le contenu :
```json
{"@orchestrateur:videocours.fr": {"user_name": "orchestrateur", "approved_at": 1719600000.0}}
```

### 5. Récap avec @mention
Bug : le récap dans les rooms privées ne mentionnait pas l'agent → le gateway le recevait
mais ne le présentait pas à l'agent (require_mention: true).
**Correction** : dans `publish()`, `recap = a + " " + recap_private` au lieu de `send_to(room, recap_private)`.

### 6. Question incluse dans le récap
Bug : `lines_full[1:]` sautait la ligne contenant la question dans le récap privé.
**Correction** : `nl.join(lines_full)` au lieu de `nl.join(lines_full[1:])`.

### 7. cycle_history (contexte des cycles)
Ajout de `format_history()` dans le state v4. Stocke les 5 derniers cycles (FIFO).
Préfixe chaque question avec le résumé :
```
📜 Cycles précédents :
  #cycle_172 "météo ?" → SOS: pluie | THREE: soleil | ...
@agent ❓ Question de Christophe : ...
```

### 8. Verrouillage vitrine
`events_default = 51` via `PUT rooms/!McOijz.../state/m.room.power_levels`.
Plus aucun agent ne peut poster dans hermes-team (M_FORBIDDEN).

### 9. Systemd
Service créé : `/etc/systemd/system/hermes-orchestrator-v4.service`.
`systemctl enable hermes-orchestrator-v4` → démarre au boot et après crash.
⚠️ Piège EnvironmentFile : le format `ORCH_TK=$(cat ...)` (littéral) fait planter
avec M_UNKNOWN_TOKEN. Solution : mettre la valeur en dur dans `Environment=`.

## VPS Dendrite (Contabo — 79.143.180.202)

- OS : Ubuntu 24.04, systemd 255
- Python : 3.13
- Hermes v3.2 orchestrateur : `/opt/dendrite/orchestrator.py` (conservé, backup)
- Hermes v4 orchestrateur : `/opt/dendrite/orchestrator_v4.py`
- State v4 : `/opt/dendrite/data/orchestrator_state_v4.json`
- Log : `/opt/dendrite/data/orch_v4.log`
- Service : `hermes-orchestrator-v4.service`
- Token : `/opt/dendrite/data/orch_token.txt`
- Rooms privées : `/opt/dendrite/data/private_rooms.json`

## VPS Chris (Hostinger — 187.124.223.135)

- Chris tourne dans un conteneur Docker `hvps-hermes-agent`
- HERMES_HOME = `/opt/data`
- Gateway relancé manuellement via `nohup hermes gateway run >> /opt/data/gateway.log 2>&1 &`
- Pairing : `/opt/data/pairing/matrix-approved.json`

## Résultat final

- 4 agents répondent en parallèle (timeout 120s)
- Récap diffusé dans vitrine + 4 rooms privées avec @mention + question + historique
- Boucle physiquement impossible (3 couches)
- Verrouillage serveur + structurel + logiciel

## Pièges SSH — upload de scripts Python sur le VPS

Le plus grand piège de la migration a été le **quoting SSH** quand on envoie des scripts
Python contenant des guillemets, des `'''`, des accolades `{}`, ou des caractères Unicode.

| Problème | Symptôme | Solution |
|---|---|---|
| Guillemets dans `bash -c "..."` | `SyntaxError: unterminated string literal` | Éviter les chaînes imbriquées ; utiliser `stdin` |
| `'''` dans un script Python | `SyntaxError: invalid syntax` | Le parseur voit la fin de la chaîne trop tôt. Utiliser `"""` ou des concaténations |
| Caractères Unicode (`\\u2753`, `\\U0001F4CB`) | `old text not found` dans le patch | Les caractères réels (`❓`, `📋`) diffèrent des échappements. Vérifier le fichier avant |
| `nohup ... &` dans SSH | Timeout (SSH bloque sur les descripteurs) | Utiliser `subprocess.Popen` dans un script Python passé via stdin |
| Heredoc + `$()` | Le `$()` est expansé par bash | Utiliser des quotes simples `'EOF'` ou stdin |

**Méthode robuste pour patcher un fichier distant :**
```bash
# 1. Écrire le script localement (fichier .py)
# 2. L'uploader via stdin
ssh root@VPS "cat > /tmp/mon_patch.py" < mon_patch.py
# 3. L'exécuter
ssh root@VPS "python3 /tmp/mon_patch.py"
```

**Méthode pour scripts courts (< 20 lignes) :**
```bash
echo 'script...' | base64 | ssh root@VPS "base64 -d | python3"
```
Ou directement via `ssh ... "python3"` avec le script en stdin (pas de quoting).
