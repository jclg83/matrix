# Approbations temporaires pilotées par `/orch`

## Contexte

Un flag stocké uniquement dans `orchestrator_v4.py` ne peut pas approuver des commandes Hermes : les approval gates appartiennent aux gateways Hermes de chaque agent, sur leurs hôtes respectifs. L’orchestrateur ne voit que les messages Matrix et l’état des cycles.

## Invariant de conception

Une future commande `orch appro` doit être **scopée**, jamais globale :

- portée : room privée Matrix concernée + session/cycle orchestré ;
- activation : uniquement après une commande explicite de Christophe dans la vitrine ;
- désactivation obligatoire : `orch reset` ;
- sécurité défensive : désactivation au redémarrage, à l’expiration du cycle, et hors room/cycle autorisé ;
- aucun impact sur Telegram, les autres rooms, les nouvelles sessions, ni les exécutions hors `/orch`.

## Brique Hermes existante (confirmée sur SOS v0.18)

Le gateway Hermes possède déjà `/yolo` : il bypass les approvals de commandes dangereuses **pour la session courante uniquement**. L'état est effacé par `/new` / `/reset`, ce qui confirme qu'une portée conversationnelle est réalisable sans modifier `approvals.mode` global.

⚠️ `/yolo` est un **toggle**, donc il n'est pas un protocole fiable pour l'orchestrateur : une répétition, un redémarrage gateway ou une action manuelle peut inverser son état. L'intégration doit employer un contrôle idempotent `on` / `off`, accepté exclusivement depuis `@orchestrateur:videocours.fr` dans la room privée autorisée.

## Architecture nécessaire

L’implémentation réelle doit coordonner :

1. le state de l’orchestrateur (activation, diffusion, reset, expiration) ;
2. un mécanisme côté **chaque gateway agent** qui reconnaît et applique une autorisation liée à cette room/session ;
3. des contrôles idempotents, consommés avant toute dispatch au modèle. Sur SOS v0.18, le chemin minimal testé est une extension strictement autorisée du handler existant : `!yolo orch on <cycle-id>` / `!yolo orch off`. Elle doit vérifier **plateforme Matrix + room privée de l’agent + émetteur `@orchestrateur:videocours.fr`** avant d’appeler `enable_session_yolo()` ou `disable_session_yolo()` ; le `/yolo` normal sans ces arguments garde son comportement de toggle ;
4. le repère vitrine `🚩 APPRO ACTIVE`, le temps restant dans `orch status`, et les messages explicites d'expiration/révocation ;
5. la validation Matrix réelle aller-retour : `orch appro` doit produire l’activation interne et `orch reset` la révocation **avant** son `/new`. Les tests unitaires ne suffisent pas à prouver que le message Matrix traverse le gateway et débloque effectivement les approval gates.

Ne jamais remplacer cela par `approvals.mode: off` global : cette option désactive les garde-fous en dehors de la conversation Matrix visée.
