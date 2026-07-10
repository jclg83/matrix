# Déploiement progressif de `orch reset` sans confirmations interactives

## But

`orch appro` doit activer l'approbation temporaire chez chaque agent déjà validé, et `orch reset` doit réinitialiser réellement la session Hermes de chaque agent déjà validé, sans afficher la confirmation `/new`, tout en restant limité à :

- plateforme `matrix` ;
- émetteur `@orchestrateur:videocours.fr` ;
- room privée exacte de l’agent.

L’agent reçoit :

```text
@agent !yolo orch on <approval_id>
@agent !yolo orch reset
```

Son handler doit :

1. pour `orch on`, activer l'approbation de session (`enable_session_yolo`) et répondre `ORCH APPRO activee` ;
2. pour `orch reset`, révoquer l’approbation de session (`disable_session_yolo`) ;
3. transformer temporairement l’événement en `/new` ;
4. appeler directement `_handle_reset_command(event)` ;
5. restaurer le texte original dans un `finally`.

Le contrôle `orch on|off|reset` doit être refusé hors de la room privée et hors de l’émetteur orchestrateur.

## Rollout obligatoire

Ne jamais ajouter un agent au dispatch global avant son test complet.

1. Ajouter d’abord un test unitaire reproduisant `orch reset` : le reset doit être appelé une fois et YOLO doit être désactivé. Constater l’échec (RED).
2. Patcher le handler de l’agent, avec une sauvegarde du fichier source.
3. Vérifier compilation + test vert (GREEN).
4. Redémarrer le gateway qui utilise précisément ce source et son `HERMES_HOME` réel.
5. Tester dans la room privée, directement et isolément : `!yolo orch on <id>`, puis `!yolo orch reset`.
6. Vérifier dans l’historique Matrix/logs que la réponse est `New session started`, sans `Confirm /new`.
7. Seulement alors ajouter l’agent à `AUTO_RESET_AGENTS` dans l’orchestrateur.

Pendant le rollout, les agents non validés doivent être journalisés comme `RESET session skipped (rollout pending)` : ne jamais leur envoyer `/new`.

## Agents Windows accessibles seulement via Chris

Quand l’agent cible est joint via Chris :

```text
SOS -> SSH root@Chris -> docker exec conteneur Chris
    -> SSH Chris avec sa clé -> tunnel local Windows :2222/:2522
```

- Les guillemets deviennent fragiles sur ce double saut. Utiliser `powershell.exe -NoProfile -EncodedCommand <UTF-16LE/base64>` pour les diagnostics, éditions et redémarrages Windows plutôt que des chaînes `cmd /c` imbriquées.
- Obtenir le vrai chemin du module avec le Python de l’agent : `import gateway.slash_commands as s; print(s.__file__)`. Ne pas supposer `site-packages` : une installation editable peut importer depuis la racine du dépôt.
- Au redémarrage, définir explicitement le `HERMES_HOME` de l’agent. Un lancement SSH sous un autre compte Windows peut démarrer un process vivant mais sans sa configuration Matrix.
- PowerShell est insensible à la casse pour les variables : éviter `$home`, qui entre en conflit avec `$HOME` en lecture seule ; utiliser `$hermesHome`.
- Avant d’arrêter un second gateway, identifier son runtime, son chemin, son profil et sa responsabilité. Deux `pythonw.exe ... gateway run --replace` peuvent être un launcher et son enfant (même heure de création, parent/child PIDs), pas deux consumers concurrents.

### Three — route validée

Route observée via Chris/VPS :

```bash
ssh -i /opt/data/ssh/hermes-nauticode-vps-187.124.223.135 root@187.124.223.135 \
  "ssh -i /root/.ssh/chris_to_windows_ed25519 -p 2222 JCLG@127.0.0.1"
```

Pour Three, le `HERMES_HOME` réel utilisé lors du redémarrage était :

```text
C:\Users\User\AppData\Local\hermes
```

### Five — précondition tunnel

Chris peut répondre `ssh Five`, mais l’alias n’est pas forcément résolu depuis tous les contextes (host vs conteneur). Demander la commande complète/résolue si besoin : HostName/IP, port, user, IdentityFile, ProxyCommand.

Route résolue observée : port tunnel `127.0.0.1:2522`, clé `/opt/data/ssh/hermes-chris-to-five`.

Avant de tenter Five, vérifier depuis le VPS Chris que le tunnel inverse écoute :

```bash
ss -ltnp | grep ':2522'
```

Si `127.0.0.1:2522` retourne `Connection refused`, arrêter le rollout Five : demander à Christophe/Five de relancer le tunnel inverse plutôt que de deviner une autre route.

## Preuves de validation

Conserver :

- sortie RED puis GREEN du test ;
- PID du gateway remplacé ;
- log `matrix connected` après redémarrage ;
- événement Matrix entrant et réponse réelle de reset ;
- logs de l’orchestrateur montrant les agents non enrôlés ignorés.

Validation finale réalisée : `orch appro` depuis la vitrine active APPRO sur `@sos`, `@three`, `@five`, `@chris`; `orch reset` envoie `@agent !yolo orch reset` aux 4 rooms privées ; les 4 agents répondent `✨ New session started!` sans `Confirm /new`.

### Règle spéciale Chris gateway

Pour tout prochain redémarrage du gateway Chris : annoncer avant exécution le motif, le diff/patch concerné, l’impact prévu et la fenêtre d’intervention ; attendre le GO explicite de Christophe. Éviter SIGKILL si le gateway a commencé son arrêt propre. Cette règle ne bloque pas les messages Matrix envoyés à Chris ni le redémarrage de l’orchestrateur Dendrite.

## Capture déterministe des réponses finales

L'orchestrateur ne doit plus traiter le premier message non filtré comme réponse officielle. Les agents Hermes produisent souvent des messages intermédiaires (`Reading skill`, `terminal`, `Working`, approvals, warnings). La capture officielle doit se faire uniquement sur un début de message strict :

```text
REPONSE FINALE SOS:
REPONSE FINALE THREE:
REPONSE FINALE FIVE:
REPONSE FINALE CHRIS:
```

Le prompt de cycle envoyé par l'orchestrateur doit rappeler explicitement : `IMPORTANT : commence ta réponse finale par exactement : REPONSE FINALE <AGENT>:`. Tout autre message en collecte est journalisé `NON FINAL` et ignoré pour le récap.

Timeouts recommandés après ce changement : `PAR_TIMEOUT = 480`, `SEQ_TIMEOUT = 480`.

## Mode économie pendant rollout

Quand tous les agents sont dans `AUTO_RESET_AGENTS`, tester depuis la **vitrine avec Christophe comme émetteur**. Un message envoyé par le token de l'orchestrateur dans la vitrine ne passe pas le garde `sender == ADMIN_USER` et peut donner un faux négatif.

Checklist :

1. Nettoyer tout ancien état de rollout (`orch_appro_sos`, `orch_appro_sos_id`) si le code a migré de `appro_sos_on` vers `appro_on`.
2. Envoyer `orch appro` depuis le compte Christophe dans la vitrine.
3. Vérifier dans les 4 rooms privées que chaque agent reçoit `@agent !yolo orch on <id>` et répond `ORCH APPRO activee`.
4. Envoyer `orch reset` depuis la vitrine.
5. Vérifier dans les 4 rooms privées que chaque agent reçoit `@agent !yolo orch reset` et répond `New session started!` ; aucun `Confirm /new` ne doit apparaître.
6. Vérifier que `orch_commands_v4.py` retourne l'action générique `appro_on`, pas une ancienne action spécifique `appro_sos_on`.

## Mode économie pendant rollout

Si le modèle actif est coûteux ou si Christophe demande explicitement l’économie :

- grouper les lectures indépendantes ;
- éviter les diagnostics bonus ;
- s’arrêter au premier vrai bloqueur externe ;
- répondre en tableau/statut court avec la prochaine action nécessaire.