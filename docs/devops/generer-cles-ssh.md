# 🔑 Générer et utiliser des clés SSH

## 🎯 Objectif

Créer une paire de clés SSH moderne, l'utiliser pour s'authentifier sur GitHub
et sur des serveurs, et l'exploiter proprement dans une pipeline CI/CD.

---

## 🧠 Rappel : comment ça marche

Une paire de clés est composée de :

| Fichier | Rôle | Partage |
|---------|------|---------|
| `~/.ssh/id_ed25519` | clé **privée** : prouve votre identité | ❌ ne quitte jamais votre poste |
| `~/.ssh/id_ed25519.pub` | clé **publique** : déposée sur les serveurs | ✅ peut être diffusée |

Le serveur vérifie que vous détenez la clé privée correspondant à une clé
publique autorisée, sans que la clé privée ne transite sur le réseau.

---

## 🛠️ Générer une clé

Utiliser **Ed25519** : plus courte, plus rapide et plus sûre que RSA.

```bash
ssh-keygen -t ed25519 -C "prenom.nom@poste-travail" -f ~/.ssh/id_ed25519
```

- `-C` : commentaire pour identifier la clé (poste, usage), pas forcément un e-mail.
- **Saisir une passphrase** : si la clé est volée, elle reste inutilisable sans elle.

Si un système ancien n'accepte pas Ed25519 :

```bash
ssh-keygen -t rsa -b 4096 -C "prenom.nom@poste-travail"
```

Vérifier les droits (SSH refuse une clé privée lisible par d'autres) :

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

---

## 🔓 Éviter de retaper la passphrase : ssh-agent

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh-add -l          # lister les clés chargées
```

---

## 🐙 Ajouter la clé à GitHub

1. Copier la clé publique :

   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```

2. GitHub → **Settings** → **SSH and GPG keys** → **New SSH key**.
3. Donner un titre explicite (poste, usage), choisir le type **Authentication Key** et coller la clé.
4. Tester :

   ```bash
   ssh -T git@github.com
   # Hi utilisateur! You've successfully authenticated, but GitHub does not provide shell access.
   ```

GitHub ne gère pas de date d'expiration sur les clés SSH : passer en revue
régulièrement la liste des clés et supprimer celles qui ne servent plus.

Sur GitLab, le chemin est **Préférences** → **SSH Keys**, avec un usage
**Authentication & Signing** et une date d'expiration.

---

## 🗂️ Plusieurs clés : le fichier `~/.ssh/config`

Une clé par usage (perso, client, serveurs) limite l'impact d'une fuite.

```text
# ~/.ssh/config
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github
    IdentitiesOnly yes

# Second compte GitHub (client) : alias d'hôte dédié
Host github-client
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_client
    IdentitiesOnly yes

Host srv-recette
    HostName 10.0.0.12
    User deploy
    IdentityFile ~/.ssh/id_ed25519_deploy
```

`IdentitiesOnly yes` évite que SSH essaie toutes les clés de l'agent
(et soit rejeté après trop de tentatives).

```bash
git clone git@github-client:organisation/projet.git
ssh srv-recette
```

---

## 🖥️ Déposer la clé sur un serveur

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub deploy@10.0.0.12
```

Puis durcir le serveur dans `/etc/ssh/sshd_config` :

```text
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```

```bash
sudo sshd -t && sudo systemctl reload ssh
```

⚠️ Garder une session ouverte pendant le test : une erreur de configuration
peut vous enfermer dehors.

---

## ✍️ Signer ses commits avec SSH

La même clé peut signer les commits (badge **Verified** sur GitHub) :

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

Sur GitHub, ajouter la clé une seconde fois avec le type **Signing Key**
(**Settings** → **SSH and GPG keys** → **New SSH key**) : une clé déclarée
uniquement pour l'authentification ne valide pas les signatures.

---

## 🔄 Utiliser une clé SSH dans GitHub Actions

Cas typique : un job de déploiement se connecte à un serveur.

1. Générer une **clé dédiée** au déploiement, sans passphrase, jamais réutilisée ailleurs :

   ```bash
   ssh-keygen -t ed25519 -C "deploy-ci-projet" -N "" -f ./deploy_key
   ```

2. Déposer `deploy_key.pub` dans `~/.ssh/authorized_keys` de l'utilisateur de déploiement.
3. Dans GitHub → **Settings** → **Environments** → `production` → **Environment secrets** :
   - `DEPLOY_SSH_KEY` : contenu de `deploy_key` ;
   - `DEPLOY_KNOWN_HOSTS` : sortie de `ssh-keyscan 10.0.0.12`.

   Un secret d'environment n'est lisible que par les jobs qui ciblent cet
   environment, qui peut lui-même exiger une approbation (**Required reviewers**)
   et n'accepter que certains tags (**Deployment branches and tags**).
4. Supprimer les fichiers `deploy_key*` du poste.

```yaml
deploy:
  if: startsWith(github.ref, 'refs/tags/v')
  runs-on: ubuntu-latest
  environment: production
  permissions:
    contents: read
  steps:
    - name: Accès SSH à l'hôte
      env:
        DEPLOY_SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
        DEPLOY_KNOWN_HOSTS: ${{ secrets.DEPLOY_KNOWN_HOSTS }}
      run: |
        install -m 700 -d ~/.ssh
        printf '%s\n' "$DEPLOY_SSH_KEY" > ~/.ssh/id_ed25519 && chmod 600 ~/.ssh/id_ed25519
        printf '%s\n' "$DEPLOY_KNOWN_HOSTS" > ~/.ssh/known_hosts
    - run: ssh deploy@10.0.0.12 "cd /opt/app && docker compose pull && docker compose up -d"
```

Points importants :

- **`known_hosts` renseigné** plutôt que `StrictHostKeyChecking no`, qui ouvre la porte à une attaque de l'homme du milieu.
- **Secret passé par `env:`** puis écrit avec `printf '%s\n'` : GitHub masque sa valeur dans les logs et les retours à la ligne de la clé sont conservés ; ne jamais l'interpoler directement dans `run:`.
- **Utilisateur de déploiement à droits limités** côté serveur, jamais `root`.
- Pour restreindre encore, préfixer la clé dans `authorized_keys` :
  `command="/opt/app/deploy.sh",no-port-forwarding,no-pty ssh-ed25519 AAAA...`

---

## ✅ Bonnes pratiques

- [ ] Ed25519 par défaut
- [ ] Passphrase sur les clés personnelles
- [ ] Une clé par usage, une clé dédiée par pipeline de déploiement
- [ ] Revue régulière des clés déposées sur GitHub (pas d'expiration automatique)
- [ ] Authentification par mot de passe désactivée sur les serveurs
- [ ] Révocation immédiate d'une clé en cas de départ ou de fuite
- [ ] Jamais de clé privée dans un dépôt Git (voir [sécurité](../security/security-best-practices.md))
