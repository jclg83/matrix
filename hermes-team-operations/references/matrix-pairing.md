# Pairing Matrix — Approbation des utilisateurs

Quand un gateway Hermes reçoit un message Matrix d'un utilisateur inconnu,
il le rejette avec `Unauthorized user: @user:server on matrix` et ne le transmet pas à l'agent.

## Comment Hermes charge le fichier de pairing

```python
from hermes_constants import get_hermes_dir
PAIRING_DIR = get_hermes_dir("platforms/pairing", "pairing")
```

`get_hermes_dir` retourne l'ancien chemin si le dossier existe déjà sur disque :

```python
home = get_hermes_home()
old_path = home / old_name          # ex: /opt/data/pairing/
if old_path.exists():
    return old_path                 # ← utilisé !
return home / new_subpath           # ex: /opt/data/platforms/pairing/ (ignoré)
```

Le fichier doit s'appeler `{platform}-approved.json`. Pour Matrix : `matrix-approved.json`.

**Piège** : si le dossier `$HERMES_HOME/pairing/` existe (même vide), c'est LUI qui est utilisé,
pas `$HERMES_HOME/platforms/pairing/`. Vérifier avec :
```python
from hermes_constants import get_hermes_dir
print(get_hermes_dir("platforms/pairing", "pairing"))
```

## Créer le fichier de pairing

```bash
# Méthode fiable : écrire via Python
ssh root@VPS "python3" <<'PYEOF'
import json
payload = {"@orchestrateur:videocours.fr": {"user_name": "orchestrateur", "approved_at": 1719600000.0}}
with open("/opt/data/pairing/matrix-approved.json", "w") as f:
    json.dump(payload, f)
print("OK")
PYEOF
```

## Redémarrer le gateway après pairing

```bash
# Container Docker (cas Chris - VPS Hostinger)
ssh root@187.124.223.135
docker exec hermes-agent-jjqz-hermes-agent-1 bash -c "nohup hermes gateway run >> /opt/data/gateway.log 2>&1 &"

# Windows (cas SOS/Three/Five)
schtasks /end /tn Hermes_Gateway
schtasks /run /tn Hermes_Gateway
```

## Vérifier

```python
from hermes_gateway.pairing import PairingStore
store = PairingStore()
print(store.is_approved("matrix", "@orchestrateur:videocours.fr"))
```
