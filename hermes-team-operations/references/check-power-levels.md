# Vérification et modification des power levels Matrix

Script à exécuter sur le VPS. **Token Christophe requis** (PL 100) — l'orchestrateur ne peut pas modifier les PL.

## Vérifier l'état actuel

```python
# check_pl.py — à exécuter sur le VPS avec le token Christophe
import json, urllib.request

TOKEN = open('/opt/dendrite/data/tokens.txt').read().strip().split('\n')[0]  # token Christophe
# ou : TOKEN = '...'  # token en dur si connu
ROOM = '%21McOijzMcBSlvlfId%3Avideocours.fr'
BASE = 'https://matrix.videocours.fr/_matrix/client/v3'

url = f'{BASE}/rooms/{ROOM}/state/m.room.power_levels'
req = urllib.request.Request(url, headers={'Authorization': 'Bearer ' + TOKEN})
data = json.loads(urllib.request.urlopen(req).read())

users = data.get('users', {})
events = data.get('events', {})

print('=== Power Levels ===')
for u, pl in sorted(users.items(), key=lambda x: x[1], reverse=True):
    print(f'  {u}: {pl}')
print(f'  m.room.message default: {events.get("m.room.message", "?")}')
```

## Appliquer la config cible

```python
# set_pl.py — à exécuter sur le VPS avec le token Christophe
import json, urllib.request

TOKEN = open('/opt/dendrite/data/tokens.txt').read().strip().split('\n')[0]  # token Christophe
ROOM = '%21McOijzMcBSlvlfId%3Avideocours.fr'
BASE = 'https://matrix.videocours.fr/_matrix/client/v3'

# 1. Lire l'état actuel
url = f'{BASE}/rooms/{ROOM}/state/m.room.power_levels'
req = urllib.request.Request(url, headers={'Authorization': 'Bearer ' + TOKEN})
data = json.loads(urllib.request.urlopen(req).read())

# 2. Appliquer la config cible
data['users']['@orchestrateur:videocours.fr'] = 51
for agent in ['@sos:videocours.fr', '@three:videocours.fr', '@five:videocours.fr', '@chris:videocours.fr']:
    data['users'][agent] = 50
# events_default reste à 50

# 3. PUT
url_put = f'{BASE}/rooms/{ROOM}/state/m.room.power_levels'
body = json.dumps(data).encode()
req2 = urllib.request.Request(url_put, data=body, method='PUT')
req2.add_header('Authorization', 'Bearer ' + TOKEN)
req2.add_header('Content-Type', 'application/json')
resp = urllib.request.urlopen(req2)
event_id = json.loads(resp.read()).get('event_id', '?')
print(f'✅ event_id: {event_id}')

# 4. Vérifier
url_get = f'{BASE}/rooms/{ROOM}/state/m.room.power_levels'
req3 = urllib.request.Request(url_get, headers={'Authorization': 'Bearer ' + TOKEN})
final = json.loads(urllib.request.urlopen(req3).read())
for u, pl in sorted(final['users'].items(), key=lambda x: x[1], reverse=True):
    print(f'  {u}: {pl}')
print(f'  events_default: {final["events"].get("m.room.message", "?")}')
```

## Config finale attendue

| Utilisateur | PL |
|---|---|
| `@christophe:videocours.fr` | 100 |
| `@orchestrateur:videocours.fr` | **51** |
| `@sos:videocours.fr` | **50** |
| `@three:videocours.fr` | **50** |
| `@five:videocours.fr` | **50** |
| `@chris:videocours.fr` | **50** |
| `m.room.message` events_default | **50** |

## Pourquoi 51/50 et pas 50/49 ?

Le design 50/49 (orchestrateur 50, agents 49) nécessitait de baisser `events_default` à 49 pour que les agents puissent parler, ce qui abaissait la barre de sécurité pour toute la room.

Le design 51/50 (orchestrateur 51, agents 50) garde `events_default` à 50, donc :
- Agents (50) ≥ events_default (50) → peuvent parler quand démutés
- Orchestrateur (51) > agents (50) → peut les muter/démuter
- `events_default` à 50 inchangé → pas de régression de sécurité
