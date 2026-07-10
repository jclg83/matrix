# Mise à jour des skills partagés — flow fork + PR

Quand un agent (ex: SOS) maintient un skill partagé dans un repo GitHub qu'il ne possède pas
(ex: `jclg83/matrix`), le push direct est bloqué si le token de l'agent n'a pas les droits
collaborateur. Pattern :

## Flow standard

1. **Fork le repo source** : `POST /repos/{owner}/{repo}/forks` vers le compte de l'agent
2. **Push les fichiers mis à jour** : `PUT /repos/{agent}/{repo}/contents/{path}` (SKILL.md + references/)
3. **Créer une PR** : `POST /repos/{owner}/{repo}/pulls` avec `head: "agent:main"`, `base: "main"`
4. **Notifier le propriétaire** : lien PR dans la vitrine Matrix, 2 clics pour merger

## Code Python de référence

```python
import urllib.request, json, base64

# Token du compte agent (ex: hermes-sos)
with open("path/to/token.txt") as f:
    token = f.read().strip()

headers = {
    "Authorization": f"Bearer {token}",
    "Accept": "application/vnd.github+json",
    "User-Agent": "Hermes-Agent"
}

# 1. Fork
data = json.dumps({"default_branch_only": True}).encode()
req = urllib.request.Request(
    f"https://api.github.com/repos/{SOURCE_OWNER}/{REPO}/forks",
    data=data, headers=headers, method="POST"
)
with urllib.request.urlopen(req) as resp:
    fork = json.loads(resp.read())  # fork['full_name'] → agent/matrix

# 2. Update SKILL.md (requires SHA from GET first)
req = urllib.request.Request(
    f"https://api.github.com/repos/{AGENT}/{REPO}/contents/{SKILL_PATH}/SKILL.md",
    headers=headers
)
with urllib.request.urlopen(req) as resp:
    sha = json.loads(resp.read())['sha']

with open("path/to/local/SKILL.md", "rb") as f:
    content = f.read()
content_b64 = base64.b64encode(content).decode()

data = json.dumps({
    "message": "Update skill to vX.Y.Z",
    "content": content_b64,
    "branch": "main", "sha": sha
}).encode()

req = urllib.request.Request(
    f"https://api.github.com/repos/{AGENT}/{REPO}/contents/{SKILL_PATH}/SKILL.md",
    data=data, headers=headers, method="PUT"
)
with urllib.request.urlopen(req) as resp:
    result = json.loads(resp.read())

# 3. Push reference files (same pattern, check existence first)
for fname in os.listdir("path/to/local/references/"):
    remote_path = f"{SKILL_PATH}/references/{fname}"
    # GET for SHA (or None if new), PUT with content + optional sha
    ...

# 4. Create PR
data = json.dumps({
    "title": "Update skill to vX.Y.Z",
    "body": "Description des changements...",
    "head": f"{AGENT}:main",
    "base": "main"
}).encode()

req = urllib.request.Request(
    f"https://api.github.com/repos/{SOURCE_OWNER}/{REPO}/pulls",
    data=data, headers=headers, method="POST"
)
with urllib.request.urlopen(req) as resp:
    pr = json.loads(resp.read())
    print(f"PR: {pr['html_url']}")
```

## Pièges

- **Token scope** : besoin de `repo` (fork + push + PR). Le token `hermes-sos` a ce scope, mais pas l'accès collaborateur au repo cible — d'où le fork.
- **SHA obligatoire** pour les updates : toujours faire un GET avant un PUT.
- **Références manquantes** : si `references/` n'existe pas dans le fork, le premier `PUT` dans ce dossier le crée automatiquement.
- **Ne pas oublier de notifier** : la PR seule ne suffit pas, Christophe doit savoir qu'elle existe.
