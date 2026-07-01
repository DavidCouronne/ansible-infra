# Gestion des Secrets avec Ansible Vault

Ce projet utilise **Ansible Vault** pour chiffrer les informations sensibles (mots de passe de connexion, identifiants du NAS, etc.) au sein du dépôt Git.

---

## 1. Principe de Fonctionnement

Les variables du projet sont séparées en deux fichiers situés dans `group_vars/all/` :
1. **[vars.yml](../group_vars/all/vars.yml)** : Contient les variables de configuration publiques et lie les variables sensibles aux variables de Vault.
2. **[vault.yml](../group_vars/all/vault.yml)** : Contient les secrets chiffrés avec Ansible Vault (préfixés par `vault_`).

*Exemple dans `vars.yml` :*
```yaml
nas_smb_password: "{{ vault_nas_smb_password }}"
```

---

## 2. La Clé de Déchiffrement (`.vault_pass`)

Pour éviter de saisir le mot de passe du coffre-fort à chaque commande, Ansible est configuré pour lire la clé dans le fichier `.vault_pass` à la racine du projet.

> [!IMPORTANT]
> Le fichier `.vault_pass` est listé dans le fichier `.gitignore`. **Il ne doit jamais être commité sur Git.**

### Génération d'une nouvelle clé (installation fraîche)
Si le fichier `.vault_pass` est absent, générez une clé aléatoire forte :
```bash
openssl rand -base64 32 > .vault_pass
chmod 600 .vault_pass
```
*Note : Si vous récupérez un dépôt existant dont le fichier `vault.yml` est déjà chiffré, vous devez récupérer la clé `.vault_pass` originale auprès de l'administrateur et la copier à la racine du projet.*

---

## 3. Commandes d'Administration de Vault

Avant de modifier les secrets, assurez-vous que votre variable d'environnement `EDITOR` est configurée dans votre terminal. C'est elle qui détermine quel éditeur s'ouvrira lors de la commande `edit`.

```bash
# Pour utiliser VS Code (recommandé si installé)
export EDITOR="code --wait"

# Pour utiliser Nano (simple et disponible par défaut)
export EDITOR="nano"
```

### Visualiser les secrets déchiffrés
Pour afficher les secrets directement dans le terminal sans les modifier :
```bash
ansible-vault view group_vars/all/vault.yml
```

### Modifier les secrets existants
Pour ouvrir et éditer les secrets dans l'éditeur de texte configuré :
```bash
ansible-vault edit group_vars/all/vault.yml
```
*Dès que vous fermez le fichier dans l'éditeur, Ansible chiffre à nouveau le fichier automatiquement.*

### Chiffrer un nouveau fichier ou écraser les secrets
Si vous devez recréer entièrement le fichier `vault.yml` :
1. Créez un fichier temporaire `/tmp/vault_temp.yml` contenant vos variables :
   ```yaml
   vault_ansible_become_password: "votre_mot_de_passe_sudo_bazzite"
   vault_nas_smb_username: "votre_utilisateur_nas"
   vault_nas_smb_password: "votre_mot_de_passe_nas"
   ```
2. Chiffrez le fichier et enregistrez-le dans `group_vars` :
   ```bash
   ansible-vault encrypt /tmp/vault_temp.yml --output group_vars/all/vault.yml
   ```
3. **Important** : Supprimez immédiatement le fichier temporaire en clair :
   ```bash
   rm /tmp/vault_temp.yml
   ```
