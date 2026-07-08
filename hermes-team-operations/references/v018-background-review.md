# v0.18 — Background Review (self-improvement moins cher)

## Qu'est-ce que c'est ?

Depuis Hermes v0.18, le **fork post-turn** (analyse après chaque message pour décider « cette info mérite-t-elle une mémoire / un skill ? ») peut être routé vers **un modèle auxiliaire moins coûteux**, au lieu d'utiliser le modèle principal à chaque fois.

Économie estimée : ~50-65% sur les forks post-turn (selon FIVE).

## Config Hermes

La clé de configuration est **`auxiliary.background_review`** (et non "self_improvement" comme on pourrait le penser).

```yaml
auxiliary:
  background_review:
    provider: "openrouter"          # "auto" = main model
    model: "google/gemini-2.5-flash-lite"  # vide = main model
    base_url: ""
    api_key: ""
    timeout: 120
```

### Via CLI

```bash
hermes config set auxiliary.background_review.provider openrouter
hermes config set auxiliary.background_review.model google/gemini-2.5-flash-lite
```

### Via le dashboard Hermes

Le paramètre `background_review` **n'apparaît pas** dans la liste des tâches auxiliaires visibles du dashboard (section "Auxiliary Tasks"). Il faut passer par le config YAML ou la CLI.

## Comportement

| Modèle auxiliaire | Effet |
|---|---|
| `auto` (même que le principal) | Rejoue la conversation complète (chaud en cache → lectures peu coûteuses) |
| Modèle différent | Rejoue un **digest compact** au lieu du transcript complet (évite le cache froid onéreux) |

La qualité est identique dans les benchmarks (capture mémoire, skills).

## ⚠️ Piège : fiabilité Gemini sur OpenRouter

**Gemini 2.5 Flash Lite fait du 503 (surcharge) de façon récurrente** sur OpenRouter. Constaté le 05/07/2026 lors de la configuration de Three.

Pour un fork post-turn qui doit tourner **après chaque message**, un modèle qui répond 503 de temps en temps fait perdre des mémoires. À éviter pour le background review.

Solution : prendre un modèle fiable d'un autre provider au même prix.

## Décision finale de l'équipe (05/07/2026)

Après 3 cycles d'évaluation collective, **Gemini abandonné pour cause de 503**. Christophe a retenu :

| Critère | Choix final |
|---|---|
| **Provider** | OpenRouter |
| **Modèle** | `openai/gpt-4.1-nano` |
| **Prix** | $0.10/M input, $0.40/M output |
| **Raison** | Même prix que Gemini, mais OpenAI fiable |
| **Note** | Chris recommande depuis le début, et il triomphe 😄 |

**Pourquoi pas Qwen3-30B-A3B** (consensus des agents à $0.048/M) : Christophe a préféré la fiabilité d'OpenAI au meilleur prix de Qwen.

### Candidats évalués (3 cycles)

Par ordre de prix :

| Modèle | Prix input/M | Prix output/M | Verdict final | Justification |
|---|---|---|---|---|
| Qwen3-30B-A3B-instruct-2507 | $0.048 | $0.19 | ✅ Bon mais écarté | Meilleur rapport Q/prix, consensus agents, Christophe préfère la fiabilité |
| Llama 3.2 3B (OpenRouter) | $0.05 | $0.34 | ❌ Trop faible | Qualité insuffisante pour décision binaire |
| Qwen3.5 Flash | $0.065 | $0.26 | ✅ Bon | Très bon, légèrement plus cher |
| **GPT-4.1-nano** | **$0.10** | **$0.40** | **✅ Vainqueur** | **Même prix que Gemini, fiable** |
| Gemini 2.5 Flash Lite | $0.10 | $0.40 | ❌ 503 | Même prix, fiable côté Google direct, mais 503 via OpenRouter |
| deepseek-chat (V4 Flash) | $0.14 | $0.28 | ✅ Bon candidat | Même provider que les agents DeepSeek |
| DeepSeek V3 | $0.27 | $1.10 | ✅ Bon | Overkill pour décision binaire |
| Claude Haiku 3.5 | $0.80 | $4.00 | ❌ Trop cher | 8× plus cher que GPT-4.1-nano |
| Grok (xAI) | $1.00+ | $2.00+ | ❌ Inadapté | 10-25× plus cher, pas de petit modèle |
| Gemini 3.5 Flash | $1.50 | $9.00 | ❌ Trop cher | Proposé par défaut par le dashboard, 15× plus cher |

**Note** : Gemini 2.5 Flash Lite reste un bon choix si l'accès se fait via **le provider Google direct** (clé Google API) plutôt qu'OpenRouter. Les 503 n'ont été constatés que sur OpenRouter.

### Installation

Three (RTX 5090, déjà en v0.18) configure en premier. Five (RTX 3090) suit après upgrade vers v0.18.

**Via CLI** (recommandé car `background_review` n'apparaît pas dans le dashboard) :
```bash
hermes config set auxiliary.background_review.provider openrouter
hermes config set auxiliary.background_review.model openai/gpt-4.1-nano
```

**Via config.yaml** :
```yaml
auxiliary:
  vision:
    provider: gemini
    model: gemini-2.5-flash-lite
  background_review:
    provider: openrouter
    model: openai/gpt-4.1-nano
```

## Piège : nom de la config

**`background_review`** est contre-intuitif. Dans le code source Hermes (`hermes_cli/config.py`) :

```python
# Background review — the post-turn self-improvement fork that decides
# whether to save a memory / patch a skill.
"background_review": {
    "provider": "auto",
    "model": "",
    ...
}
```

Si on cherche "self_improvement" ou "post_turn" dans la config, on ne trouve rien. Il faut chercher `background_review`.
