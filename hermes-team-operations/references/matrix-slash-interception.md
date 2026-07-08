# Matrix Slash Interception

## Problème

Dans une room Matrix, les messages commençant par `/` sont interprétés à **deux niveaux** :

### 1. Côté Hermes — interception par les agents

L'adapter Hermes Matrix interprète `/` comme une commande slash Hermes. Résultat : quand Christophe envoie `/orch status`, les agents Hermes (SOS, Three, Five, Chris) interceptent le `/` et répondent `Unknown command /orch` avant que l'orchestrateur ne le voie.

L'orchestrateur **reçoit** le message (il utilise l'API Matrix directement, pas l'adapter Hermes), mais les agents génèrent du bruit et des interruptions.

### 2. Côté Element — interception par le client

Le client Element Desktop/Web intercepte lui aussi tout message commençant par `/` comme une commande client interne. Il affiche :

> Commande non reconnue : /orch reset
> Vous pouvez utiliser /help pour obtenir la liste des commandes disponibles.
> Vouliez-vous envoyer un message ?

Et propose « Annuler » ou « Envoyer ». Si l'utilisateur clique « Envoyer », le message `/orch reset` est envoyé comme texte brut dans la room — les agents le voient et essaient d'y répondre, générant des réponses « commande inconnue ».

## Solutions

### Pour Hermes : envoyer sans slash

L'orchestrateur accepte `orch status` (sans `/`) en plus de `/orch status`. Christophe peut envoyer `orch status` comme message normal, non interprété comme commande slash par les agents Hermes.

### Pour Element : double slash ou sans slash

- **Double slash** : taper `//orch reset` — le premier `/` échappe le second, Element envoie `/orch reset` en texte brut (que l'orchestrateur traite normalement).
- **Sans slash** : taper simplement `orch reset` — plus simple, pas d'interception ni par Element ni par Hermes, et l'orchestrateur l'accepte.

**Recommandation** : utiliser la forme sans slash (`orch reset`, `orch status`, etc.) qui évite les deux niveaux d'interception.

## Implémentation côté orchestrateur

L'orchestrateur v4 (ligne 410) accepte les trois formes :
```python
if room == VITRINE and sender == ADMIN_USER and \
   (body.startswith("/orch") or body.startswith("orch ") or body == "orch"):
```

## Commandes de patch historiques (30/06/2026)

### orch_commands.py

```bash
ssh root@79.143.180.202 "sed -i 's/if not body.startswith(\"\/orch\"):/if not (body.startswith(\"\/orch\") or body.startswith(\"orch\")):/' /opt/dendrite/orch_commands.py"
```

Avant :
```python
if not body.startswith("/orch"):
    return None
```

Après :
```python
if not (body.startswith("/orch") or body.startswith("orch")):
    return None
```

### orchestrator.py

```bash
ssh root@79.143.180.202 "sed -i \"s/body.strip().startswith('\/orch')/(body.strip().startswith('\/orch') or body.strip().startswith('orch '))/\" /opt/dendrite/orchestrator.py"
```

Avant (lignes 207 et 229) :
```python
if sender == ADMIN_USER and body.strip().startswith('/orch'):
```

Après :
```python
if sender == ADMIN_USER and (body.strip().startswith('/orch') or body.strip().startswith('orch ')):
```

## Vérification

```bash
ssh root@79.143.180.202 "grep -n 'startswith.*orch' /opt/dendrite/orch_commands.py /opt/dendrite/orchestrator.py"
```

Doit montrer les 3 lignes patchées (1 dans orch_commands.py, 2 dans orchestrator.py).
