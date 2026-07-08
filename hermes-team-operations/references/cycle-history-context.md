# cycle_history — Contexte des cycles pour les agents

Ajouté le 05/07/2026 pour que les agents gardent le fil entre les cycles.
Refondu le 06/07/2026 : suppression de la troncature 40 caractères + réorganisation `question_text`.

## Principe

À chaque nouvelle question, l'orchestrateur inclut le résumé des 5 derniers cycles dans le message envoyé à l'agent dans sa room privée. L'agent voit immédiatement ce qui a été dit avant, sans avoir à fouiller l'historique de sa room.

**Depuis le 06/07/2026**, le contexte est placé APRÈS la question (pas avant), avec un label explicite « Contexte (utilise ces cycles precedents) ». Ce repositionnement règle un problème d'interprétation : les agents traitaient l'historique comme un préambule informatif de l'orchestrateur plutôt que comme leur contexte de travail.

Format actuel reçu par les agents :
```
@agent:videocours.fr ❓ Question de Christophe : <question>

📜 Contexte (utilise ces cycles precedents) :
  ▶ #297562 ❓ <question cycle précédent en entier>
    [SOS] <réponse SOS en entier>
    [THREE] <réponse THREE en entier>
    [FIVE] <réponse FIVE en entier>
    [CHRIS] <réponse CHRIS en entier>

(Reponds dans cette room. Une seule reponse.)
```

## Implémentation

### State v4
```json
"cycle_history": [
  {"id": "cycle_1720000000", "question": "météo demain ?",
   "responses": {"@sos:...": "pluie", "@three:...": "soleil", ...}},
  ...
]
```
FIFO, max 5. Inséré dans `publish()` juste avant le cleanup :
```python
entry = {"id": st["cycle_id"], "question": st["current_question"],
         "responses": dict(st["responses"])}
st.setdefault("cycle_history", []).append(entry)
st["cycle_history"] = st["cycle_history"][-5:]
```

### format_history() — version actuelle (06/07/2026)

**Plus de troncature.** Questions et réponses passent en texte intégral.
Chaque cycle est un bloc multi-lignes avec un agent par ligne :

```python
def format_history(history):
    if not history:
        return ""
    lines = ["\U0001F4DC Cycles precedents :"]
    for h in history[-5:]:
        q = h.get("question", "?")
        lines.append("  \u25B6 #" + str(h.get("id", "?"))[-6:] + " \u2753 " + q)
        for a, resp in h.get("responses", {}).items():
            if resp is None:
                lines.append("    [" + short(a) + "] (timeout)")
            else:
                lines.append("    [" + short(a) + "] " + resp)
        lines.append("")
    return "\n".join(lines) + "\n\n"
```

**Ancien format (obsolète)** — tronquait questions à 40 car. et réponses à 40 car. sur une seule ligne, illisible :
```
📜 Cycles precedents :
  #297562 je fais une vérification orch reset (san -> SOS: C... | THREE: J... | FIVE: ✅... | CHRIS: ✅...
```

### question_text() — version actuelle (06/07/2026)

Le contexte est placé APRÈS la question (inversé par rapport à la v1), avec label explicite :

```python
def question_text(agent, q, st):
    ctx = format_history(st.get("cycle_history", []))
    body = agent + " \u2753 Question de Christophe : " + q
    if ctx:
        body += "\n\n\U0001F4DC Contexte (utilise ces cycles precedents) :\n" + ctx
    body += "\n(Reponds dans cette room. Une seule reponse.)"
    return body
```

**Pourquoi le contexte après la question ?** Quand l'historique était avant la mention `@agent`, certains agents (FIVE, parfois SOS) le traitaient comme un bloc informatif de l'orchestrateur et ne l'intégraient pas à leur contexte de travail. Avec le label « Contexte (utilise ces cycles precedents) » et le placement après la question, plus d'ambiguïté.

## `/orch reset` et cycle_history

`/orch reset` vide `cycle_history` ET envoie `@agent /new` à chaque agent dans sa room privée :

```python
elif action == "reset":
    # ... reset de l'état du cycle ...
    st["cycle_history"] = []
    # Reset sessions des agents
    for a, rid in PRIVATE_ROOMS.items():
        try:
            send_to(rid, a + " /new")
        except:
            pass
    save_state(st)
```

L'@mention est obligatoire car les agents ont `require_mention: true`.

## Pièges

- **Copie de `st["responses"]` avant cleanup** : utiliser `dict(st["responses"])` pour
  capturer une copie, pas une référence. Sinon le cleanup `st["responses"] = {}` vide aussi l'historique.
- **Taille** : les réponses complètes sans troncature peuvent être volumineuses (4000 chars/réponse × 4 agents × 5 cycles = jusqu'à 80K chars). Les LLM modernes (128K contexte) gèrent ça sans problème.
- **État initial** : les premiers cycles sont vides. Au bout de 5 cycles, l'historique est complet.
- **Changement de sujet** : l'historique des 5 derniers cycles peut parasiter les agents. Solution : `/orch reset`.
- **Redémarrage orchestrateur requis** : le script tourne en continu. Après une modification de code (format_history, question_text), un `systemctl restart hermes-orchestrator-v4` est nécessaire. Les modifs de constantes (SEQ_TIMEOUT, MAX_CHARS) sont prises en compte directement.
- **Doublon systemd** : ne pas relancer manuellement via `nohup ... &` si le service systemd est actif → 2 instances = messages en double. Utiliser `systemctl restart` ou tuer d'abord le service.
