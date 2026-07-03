# PayOS — Vault AppRole : injection sécurisée du secret-id

Last Updated: 2026-06-03

---

## Table of contents

1. [Problème](#1-problème)
2. [Solution : Docker secrets + `${file:...}`](#2-solution--docker-secrets--file)
3. [Étape 1 — Créer le fichier secret sur l'hôte](#3-étape-1--créer-le-fichier-secret-sur-lhôte)
4. [Étape 2 — Configurer bootstrap.json](#4-étape-2--configurer-bootstrapjson)
5. [Étape 3 — Configurer docker-compose.yml](#5-étape-3--configurer-docker-composeyml)
6. [Étape 4 — Vérifier le démarrage](#6-étape-4--vérifier-le-démarrage)
7. [Variante : Docker Swarm (external secret)](#7-variante--docker-swarm-external-secret)
8. [Comparaison des approches](#8-comparaison-des-approches)
9. [Règles de sécurité](#9-règles-de-sécurité)

---

## 1. Problème

L'authentification AppRole de HashiCorp Vault nécessite deux credentials :

| Credential | Sensibilité | Recommandation |
|---|---|---|
| `role-id` | Semi-public — identifie l'application | Peut figurer dans `bootstrap.json` |
| `secret-id` | **Sensible** — équivalent d'un mot de passe | Ne doit jamais apparaître dans un fichier versionné |

Stocker le `secret-id` en clair dans `bootstrap.json` expose le credential dans Git et dans les images Docker.  
Le passer en variable d'environnement le rend visible via `docker inspect` et dans les logs d'orchestration.

---

## 2. Solution : Docker secrets + `${file:...}`

PayOS supporte la syntaxe `${file:/chemin/vers/fichier}` dans tous les champs de `bootstrap.json`. Au démarrage, le runtime lit le fichier spécifié et substitue son contenu — exactement comme une variable d'environnement, mais depuis un fichier.

Docker secrets monte le fichier dans le conteneur sous forme de **tmpfs** (mémoire RAM) :
- Invisible dans `docker inspect`
- Non écrit sur disque
- Supprimé à l'arrêt du conteneur

```
hôte : ./vault-secret-id.txt  ──► conteneur : /run/secrets/vault_secret_id  (tmpfs)
                                                        │
                                              PayOS lit au démarrage
                                              via ${file:/run/secrets/vault_secret_id}
```

---

## 3. Étape 1 — Créer le fichier secret sur l'hôte

Créer le fichier contenant le `secret-id` à côté du `docker-compose.yml` :

```bash
# Linux / macOS
echo -n "s2-VOTRE-SECRET-ID-ICI" > ./vault-secret-id.txt
chmod 600 ./vault-secret-id.txt
```

```powershell
# Windows
"s2-VOTRE-SECRET-ID-ICI" | Set-Content ./vault-secret-id.txt -NoNewline
```

> **Important :** Ajouter ce fichier dans `.gitignore` pour éviter tout commit accidentel.
>
> ```
> # .gitignore
> vault-secret-id.txt
> *.secret
> ```

---

## 4. Étape 2 — Configurer bootstrap.json

Le `bootstrap.json` est baked dans l'image Docker (ne change pas pendant la vie du conteneur). Il contient un placeholder, pas la valeur réelle :

```json
{
  "secret-service": {
    "configuration": {
      "type": "vault",
      "address": "https://vault.example.com:8200",
      "role-id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "secret-id": "${file:/run/secrets/vault_secret_id}",
      "approle-mount": "approle",
      "kv-mount": "secret",
      "timeout": 10
    }
  }
}
```

Le chemin `/run/secrets/vault_secret_id` est le point de montage Docker standard. Le nom de fichier (`vault_secret_id`) doit correspondre au nom du secret déclaré dans `docker-compose.yml` (voir étape suivante).

---

## 5. Étape 3 — Configurer docker-compose.yml

```yaml
secrets:
  vault_secret_id:
    file: ./vault-secret-id.txt     # chemin relatif au docker-compose.yml, sur l'hôte

services:
  payos:
    image: payos:latest
    secrets:
      - vault_secret_id             # monté à /run/secrets/vault_secret_id dans le conteneur
```

**Explication des entrées :**

- `secrets.vault_secret_id.file` : chemin du fichier sur **la machine hôte**, relatif au `docker-compose.yml`. Docker lit ce fichier au démarrage du conteneur.
- `services.payos.secrets` : liste des secrets à monter dans ce conteneur. Docker crée un fichier tmpfs à `/run/secrets/<nom_du_secret>` accessible depuis le processus PayOS.

---

## 6. Étape 4 — Vérifier le démarrage

Au démarrage, PayOS résout le token `${file:...}` et tente de se connecter à Vault. En cas de succès, les logs indiquent :

```
INFO  VaultSecretProvider - Vault AppRole authentication successful
INFO  VaultSecretProvider - Secret provider ready [mount=secret, address=https://vault.example.com:8200]
```

En cas d'erreur de lecture du fichier :

```
IllegalStateException: Config key 'secret-id': cannot read secret file '/run/secrets/vault_secret_id': ...
```

→ Vérifier que `secrets:` est bien déclaré dans le `docker-compose.yml` et que le service y est attaché.

En cas de credentials Vault invalides :

```
IllegalStateException: Vault AppRole login failed: 403 Forbidden
```

→ Vérifier le contenu du fichier `vault-secret-id.txt` sur l'hôte.

---

## 7. Variante : Docker Swarm (external secret)

En production sur Docker Swarm, le secret est stocké dans la base de données chiffrée de Swarm — aucun fichier sur l'hôte.

**Créer le secret dans Swarm :**

```bash
echo -n "s2-VOTRE-SECRET-ID-ICI" | docker secret create vault_secret_id -
```

**Modifier docker-compose.yml :**

```yaml
secrets:
  vault_secret_id:
    external: true                  # Swarm gère le secret, pas de fichier hôte

services:
  payos:
    image: payos:latest
    secrets:
      - vault_secret_id
```

Le point de montage dans le conteneur reste identique : `/run/secrets/vault_secret_id`. Aucune modification de `bootstrap.json` n'est nécessaire.

---

## 8. Comparaison des approches

| Approche | Visible dans `docker inspect` | Dans Git risqué | Recommandation |
|---|---|---|---|
| Valeur en clair dans `bootstrap.json` | Non | **Oui** | ❌ Jamais en production |
| Variable d'environnement `${VAR}` | **Oui** | Non | ⚠️ Acceptable en développement |
| Docker secret + `${file:...}` (compose) | Non | Non | ✅ Recommandé |
| Docker Swarm secret (external) | Non | Non | ✅ Recommandé en production |

---

## 9. Règles de sécurité

- Ne jamais committer `vault-secret-id.txt` — toujours dans `.gitignore`.
- Ne jamais bake le `secret-id` dans l'image Docker (`docker build --build-arg` est visible dans l'historique de l'image).
- Utiliser un `secret-id` à durée de vie limitée et le rotationner régulièrement depuis Vault :
  ```bash
  vault write -f auth/approle/role/payos/secret-id
  ```
- En production, préférer Docker Swarm secrets ou un orchestrateur avec gestion native des secrets (Kubernetes Secrets with encryption at rest, AWS Secrets Manager, Azure Key Vault).
- `tls-skip-verify: true` est interdit en production — ne jamais désactiver la vérification TLS vers Vault.
