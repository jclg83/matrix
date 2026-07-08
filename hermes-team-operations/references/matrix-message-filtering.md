# Règles de filtrage des messages Matrix pour les agents

## Règle critique (v1.9.0)

Chaque agent Hermes (SOS, Three, Five, Chris) doit ignorer TOUS les messages Matrix de la room **sauf** :

1. Les messages de **Christophe** (`@christophe:videocours.fr`)
2. Les messages de **l'orchestrateur** (`@orchestrateur:videocours.fr`) qui **mentionnent l'agent** (contiennent `@agent:videocours.fr`)

→ **Les messages des autres agents = silence total.**

## Pourquoi

Sans cette règle, les agents se répondent entre eux et génèrent des boucles de conversation. L'orchestrateur est le seul chef d'orchestre — les agents ne dialoguent qu'avec lui et Christophe.

## Implémentation

Cette règle est intégrée dans le prompt de chaque agent. Elle se traduit par :
- Ne pas répondre aux messages de `@sos:videocours.fr`, `@three:videocours.fr`, `@five:videocours.fr`, `@chris:videocours.fr` (sauf orchestrateur)
- Ne traiter que les messages de `@christophe:videocours.fr` OU `@orchestrateur:videocours.fr` contenant sa propre mention
- Ignorer les messages système, les joins/leaves, les réactions

## Configuration Hermes

Pour éviter les interruptions par les messages entrants (`⚡️ Interrupting current task`), ajouter dans la section `matrix` du `config.yaml` :

```yaml
matrix:
  streaming:
    interrupt_notifications: false
```

Ce paramètre empêche les notifications d'interruption de polluer la room quand un agent reçoit un message pendant qu'il travaille.
