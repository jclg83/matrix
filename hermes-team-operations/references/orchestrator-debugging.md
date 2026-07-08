# Debugging orchestrateur v4

## Fichiers importants

| Fichier | Chemin | Contenu |
|---|---|---|
| Code | `/opt/dendrite/orchestrator_v4.py` | Le script Python |
| Log | `/opt/dendrite/data/orch_v4.log` | Log rotatif |
| State | `/opt/dendrite/data/orchestrator_state_v4.json` | État persistant |
| Rooms | `/opt/dendrite/data/private_rooms.json` | Mapping agent ↔ room privée |
| Token | `/opt/dendrite/data/orch_token.txt` | Token Matrix de l'orchestrateur |

## Commandes de diagnostic

```bash
# État actuel
cat /opt/dendrite/data/orchestrator_state_v4.json | grep -E 'cycle_status|current_question|cycle_id'

# Vérifier que l'orchestrateur tourne
ps aux | grep orchestrator_v4 | grep -v grep

# Compter les erreurs HTTP (0 = tout va bien)
grep -c 'HTTP\|REQ ERROR' /opt/dendrite/data/orch_v4.log

# Voir les cycles récents
grep 'CYCLE START\|CYCLE END\|TIMEOUT' /opt/dendrite/data/orch_v4.log | tail -20

# Voir les réponses collectées
grep 'RÉPONSE' /opt/dendrite/data/orch_v4.log | tail -10

# Voir les hors-cycles (agents qui répondent hors sollicitation)
grep 'HORS CYCLE' /opt/dendrite/data/orch_v4.log | tail -10

# Voir si le récap a été diffusé (pas de log explicite, mais CYCLE END appelle publish())
grep 'CYCLE END' /opt/dendrite/data/orch_v4.log

# Log complet (dernières lignes)
cat /opt/dendrite/data/orch_v4.log | tail -50

# Vérifier les power levels de la vitrine
curl -s -H "Authorization: Bearer $(cat /opt/dendrite/data/orch_token.txt)" \
  "https://matrix.videocours.fr/_matrix/client/v3/rooms/!McOijzMcBSlvlfId:videocours.fr/state/m.room.power_levels" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print('events_default:', d.get('events_default')); print('users:', json.dumps(d.get('users',{}), indent=2))"

# Redémarrage orchestrateur
ssh root@79.143.180.202 "kill \$(pgrep -f orchestrator_v4.py) 2>/dev/null; sleep 1"
ssh root@79.143.180.202 "cd /opt/dendrite && ORCH_TK=\$(cat /opt/dendrite/data/orch_token.txt) nohup python3 -u orchestrator_v4.py >> data/orch_v4.log 2>&1 &"
```

## Log interprété

### Format des lignes
```
[HH:MM:SS] MSG [vitrine] @christophe:videocours.fr: <texte>
  → message vu par l'orchestrateur dans la vitrine

[HH:MM:SS] MSG [SOS] @sos:videocours.fr: <texte>
  → message vu dans la room privée orch-sos (réponse d'agent)

[HH:MM:SS] RÉPONSE CHRIS (169 chars)
  → réponse collectée et enregistrée dans le state

[HH:MM:SS] CYCLE START par | agents=SOS,THREE,... | q=<question>
  → début de cycle, question envoyée aux rooms privées

[HH:MM:SS] CYCLE END | complete | 4/4 réponses
  → cycle terminé, publish() appelé (récap envoyé vitrine + rooms privées)

[HH:MM:SS] HORS CYCLE: SOS - ignoré
  → agent répond hors cycle, message ignoré + loggé (anti-boucle)

[HH:MM:SS] System msg de FIVE - ignoré
  → message système Hermes filtré (📬, ⚡, etc.)
```

### Pas de log = pas d'erreur
La fonction `send_to()` attrape toutes les exceptions (`urllib.error.HTTPError`, `Exception`) et les logge. Si le log n'a pas de `HTTP` ou `REQ ERROR`, tous les messages partent.

### Threads Matrix et PL
Si des agents apparaissent dans la vitrine malgré PL 50 < events_default 51, c'est que les threads Matrix contournent le verrouillage PL. Vérifier la config Dendrite :

```bash
curl -s -H "Authorization: Bearer $(cat /opt/dendrite/data/orch_token.txt)" \
  "https://matrix.videocours.fr/_matrix/client/v3/rooms/!McOijzMcBSlvlfId:videocours.fr/state/m.room.power_levels" | \
  python3 -c "
import sys, json
d = json.load(sys.stdin)
print('events_default:', d.get('events_default'))
print()
# Voir si des utilisateurs ont un PL modifié dans le thread
for k,v in sorted(d.get('users',{}).items()):
    print(f'  {k}: PL {v}')
"
```

## Scénarios typiques

| Symptôme | Cause probable | Solution |
|---|---|---|
| CYCLE END mais agents disent ne pas voir le récap | Le récap part vers la room privée avec `@mention`, mais l'agent `require_mention: true` le capte et le traite. Le récap est bien envoyé — vérifier le log. | Vérifier qu'il n'y a pas d'erreur HTTP. Le récap est informatif, l'agent doit l'ignorer. |
| Cycle sans réponse (0/4) | `allowed_rooms` mal configuré, gateway agent down, ou token expiré | Vérifier le gateway de chaque agent |
| Agents postent dans la vitrine | Thread Matrix contourne le PL | Vérifier PL du thread ou config Dendrite |
| HORS CYCLE en cascade après un récap | Agents qui répondent au récap malgré `(informatif)` | Anti-boucle couche 3 fonctionne — c'est normal |
