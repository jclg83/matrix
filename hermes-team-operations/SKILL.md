---
name: hermes-team-operations
description: Regles de fonctionnement de l equipe Hermes - architecture vitrine + rooms privees (v4), cycles collecte->diffusion, anti-boucle structurel.
version: 2.1.0
metadata:
  hermes:
    tags: [team, orchestration, rules, orchestrator-v4, commands, vitrine]
---

# Hermes Team Operations - v4

## Architecture v4 : vitrine + rooms privees

**Anti-boucle physique** : les agents ne peuvent JAMAIS poster dans la room commune.



### Cycle : 2 phases (collecte -> diffusion)

1. **Phase 1 - Collecte** : Christophe pose une question. L'orchestrateur l'envoie aux agents dans leurs rooms privees (parallele, timeout 120s). Chaque agent repond dans sa room privee.
2. **Phase 2 - Diffusion** : L'orchestrateur publie le recap complet dans la vitrine ET envoie le recap dans chaque room privee.
3. **Contexte historique** : chaque question inclut le resume des 5 derniers cycles.
4. **File d'attente** : questions stockees si posees pendant une collecte (expirent 5 min).

## Commandes /orch v4

| Commande | Effet |
|---|---|
| /orch status | Cycle, mode, etat, reponses x/4 |
| /orch stop | Arrete le cycle |
| /orch close | Close la collecte, recap partiel |
| /orch pause | Suspend les cycles |
| /orch resume | Reactive |
| /orch mode parallel|sequential | Bascule mode |
| /orch rooms | Liste les rooms privees |
| /orch disable @agent ou all | Retire un agent |
| /orch enable @agent ou all | Reintegre |
| /orch queue | Affiche la file |
| /orch clearqueue | Vide la file |
| /orch reset | Reset complet |
| /orch log [N] | N dernieres lignes de log |
| /orch help | Liste complete |

## Anti-boucle - 3 couches

| Couche | Garantie |
|---|---|
| 1. Serveur (events_default=51) | L'agent ne peut JAMAIS poster dans vitrine |
| 2. Structurel (4 rooms privees) | Un agent ne voit pas les messages bruts des autres |
| 3. Logiciel (hors-cycle ignore) | Zero propagation |

## Configuration agent (v4)



## Piges techniques

- **Interception slash** : envoyer orch status (sans /)
- **Token orchestrateur** : /opt/dendrite/data/orch_token.txt
- **Arret urgence** : ssh root@79.143.180.202 kill orchestrator_v4.py
- **Redemarrage** : cd /opt/dendrite && ORCH_TK= nohup python3 -u orchestrator_v4.py
- **Logs** : grep recap /opt/dendrite/data/orch_v4.log | tail -10
- **Systemd** : systemctl enable hermes-orchestrator-v4 && systemctl start
- **Pairing** : Si Unauthorized user -> creer matrix-approved.json
- **Gateway Chris (Docker)** : docker exec hermes-agent-jjqz-hermes-agent-1 nohup hermes gateway run
