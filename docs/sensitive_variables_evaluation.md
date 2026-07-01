# Évaluation des Variables Sensibles et Intégration Vault

Ce document évalue la sensibilité des variables de configuration utilisées dans le projet **ansible-infra** et propose un plan pour sécuriser celles qui sont actuellement stockées en clair, notamment l'adresse IP du NAS (`nas_ip`).

---

## 1. Cartographie et Analyse de Sensibilité des Variables

Le tableau ci-dessous recense les variables clés du projet, leur état actuel de sécurité, leur niveau de sensibilité et l'action recommandée.

| Variable | Fichier Source | État Actuel | Sensibilité | Description | Action Recommandée |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `vault_ansible_become_password` | `vault.yml` | **Chiffré** | **Très Élevée** | Mot de passe administrateur (sudo) de la machine Bazzite. | Conserver dans Vault. |
| `vault_nas_smb_password` | `vault.yml` | **Chiffré** | **Élevée** | Mot de passe de l'utilisateur du NAS Synology. | Conserver dans Vault. |
| `vault_nas_smb_username` | `vault.yml` | **Chiffré** | **Moyenne** | Identifiant de connexion au NAS Synology. | Conserver dans Vault. |
| `nas_ip` | `vars.yml` | **En clair** | **Moyenne-Faible** | Adresse IP locale du NAS (`192.168.1.8`). Dévoile l'architecture réseau interne. | **À déplacer dans Vault** (voir section 2). |
| `ansible_host` | `inventory.ini` | **En clair** | **Moyenne-Faible** | Adresse IP locale de la machine cible Bazzite. | Déplacer dans un fichier de variables d'hôte (voir section 3). |
| `ansible_user` | `inventory.ini` / `jellyfin.yml` | **En clair** | **Faible** | Nom d'utilisateur SSH de la cible (`famille`). | Centraliser dans l'inventaire et supprimer des playbooks. |
| `ansible_ssh_private_key_file` | `inventory.ini` | **En clair** | **Moyenne** | Chemin local de la clé SSH privée macOS. | Conserver en clair car le chemin est générique (`~/.ssh/...`). |

---

## 2. Guide Pratique : Déplacer `nas_ip` dans Ansible Vault

Pour masquer l'adresse IP du NAS Synology (`192.168.1.8`) et la rendre dynamique via le Vault, suivez ces étapes :

### Étape 1 : Ajouter l'IP dans le coffre-fort `vault.yml`
1. Configurez votre éditeur de texte dans le terminal :
   ```bash
   export EDITOR="code --wait"  # ou "nano"
   ```
2. Ouvrez le fichier de secrets chiffrés :
   ```bash
   ansible-vault edit group_vars/all/vault.yml
   ```
3. Ajoutez la variable `vault_nas_ip` à la fin du fichier :
   ```yaml
   vault_nas_ip: "192.168.1.8"
   ```
4. Enregistrez et fermez l'éditeur. Le fichier sera rechiffré automatiquement.

### Étape 2 : Référencer le secret dans `vars.yml`
Modifiez le fichier public **[vars.yml](../group_vars/all/vars.yml)** pour remplacer la valeur en dur par une liaison vers Vault :
```yaml
# Remplacer :
# nas_ip: "192.168.1.8"

# Par :
nas_ip: "{{ vault_nas_ip }}"
```

Désormais, l'adresse IP est masquée dans le code source public du dépôt Git et injectée uniquement au moment de l'exécution d'Ansible via le fichier `.vault_pass`.

---

## 3. Autres Recommandations de Sécurité

### A. Sécuriser les adresses IP d'hôtes (`ansible_host`)
L'adresse IP de votre machine Bazzite (`192.168.1.106`) est écrite en dur dans le fichier **[inventory.ini](../inventory.ini)**.
* **Problème** : L'inventaire est généralement commité, ce qui expose l'IP sur GitHub.
* **Solution** : Vous pouvez chiffrer l'adresse IP de l'hôte en utilisant des variables spécifiques d'hôte (`host_vars`).
  1. Créez un dossier `host_vars/bazzite-salon/` à la racine.
  2. Créez un fichier chiffré `vault.yml` dans ce dossier contenant :
     ```yaml
     vault_ansible_host: "192.168.1.106"
     ```
  3. Liez-la dans `host_vars/bazzite-salon/vars.yml` :
     ```yaml
     ansible_host: "{{ vault_ansible_host }}"
     ```
  4. Simplifiez `inventory.ini` :
     ```ini
     [bazzite_hosts]
     bazzite-salon
     ```

### B. Suppression des doublons de variables utilisateur
Dans le playbook **[jellyfin.yml](../playbooks/jellyfin.yml)**, la variable `ansible_user: "famille"` est définie localement.
* **Recommandation** : Supprimez cette variable du playbook. L'utilisateur Ansible doit être défini de manière centralisée dans l'inventaire ou dans `group_vars/all/vars.yml`, évitant ainsi d'avoir à modifier les playbooks si vous changez d'utilisateur cible.
