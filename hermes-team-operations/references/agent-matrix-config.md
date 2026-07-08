# Configuration Matrix pour les agents Hermes

## Template complet

```yaml
matrix:
  enabled: true
  user_id: "@agent:videocours.fr"
  server_url: "https://matrix.videocours.fr"
  access_token: "<depuis /opt/dendrite/data/tokens.txt>"
  allowed_rooms: "!McOijzMcBSlvlfId:videocours.fr"
  require_mention: false
```

## Points critiques

### `require_mention: false` — obligatoire

L'orchestrateur mentionne l'agent (`@agent:videocours.fr`) dans son message de tour de parole. Avec `require_mention: true`, l'agent peut répondre, MAIS :
- Si l'orchestrateur envoie un message de statut (sans mention), l'agent l'ignore
- Si des messages système arrivent, l'agent les ignore
- Comportement plus fiable avec `false` : l'agent traite TOUS les messages de la room et laisse l'anti-boucle (orchestrateur) filtrer les hors-tour

### `server_url` — HTTPS obligatoire

L'orchestrateur utilise `https://matrix.videocours.fr` (pas `localhost:8008`). Les agents doivent faire de même.

### `allowed_rooms` — restreindre à la room de l'orchestrateur

Évite que l'agent ne réponde dans d'autres rooms Matrix. Mettre `""` autorise toutes les rooms (non recommandé).

### `streaming.interrupt_notifications: false` — anti-bruit

Empêche les messages `⚡ Interrupting current task` de polluer la room quand l'orchestrateur ping un agent. Sans cette option, chaque ping de l'orchestrateur déclenche une cascade d'interruptions visibles dans la room, que l'orchestrateur filtre côté serveur (via `is_system_msg()`) mais qui restent du bruit visuel.

```yaml
matrix:
  streaming:
    interrupt_notifications: false
```

> Placé sous `matrix:` (même niveau que `enabled`, `homeserver_url`, etc.), PAS sous `display:`.

### Règle de filtrage Matrix — silence agent

**Règle critique pour les agents Hermes dans la room d'équipe :**

- ✅ Répondre UNIQUEMENT aux messages de **Christophe** (`@christophe:videocours.fr`)
- ✅ Répondre UNIQUEMENT aux messages de l'**orchestrateur** (`@orchestrateur:videocours.fr`) qui **mentionnent** l'agent
- ❌ Ignorer TOUS les messages des autres agents (`@sos`, `@three`, `@five`, `@chris`) — silence total
- ❌ Ne pas accuser réception, ne pas commenter, ne pas réagir aux messages des autres agents

Cette règle est appliquée au niveau de l'agent lui-même (discipline interne), en complément de l'anti-boucle de l'orchestrateur (qui ignore les messages hors-tour côté serveur). Double protection.

## Vérification

Pour vérifier que Matrix est actif :
```bash
grep -A 10 "matrix:" ~/AppData/Local/hermes-sos/config.yaml
```

Pour vérifier la connexion Matrix dans les logs Hermes :
```bash
grep -i "matrix" ~/AppData/Local/hermes-sos/logs/*.log | tail -20
```

## Tokens

Les tokens des agents sont dans `/opt/dendrite/data/tokens.txt` sur le VPS.
Chaque agent a son propre token — ne pas partager un token entre agents.
