# Configuration Initiale et Dépannage SSH & Sudo

Ce guide détaille la configuration de votre machine de contrôle (macOS) et de la machine cible (Bazzite/Fedora Atomic), ainsi que la résolution des problèmes courants liés à la connexion SSH et à l'escalade de privilèges (sudo).

---

## 1. Configuration de la Machine de Contrôle (macOS)

Ansible est installé dans un environnement virtuel Python (`venv`) pour isoler les dépendances et éviter de polluer l'installation Python système de macOS.

### Prérequis Homebrew
Installez les outils de base requis :
```bash
brew install python openssl sshpass
```
*Note : `sshpass` est requis si vous devez utiliser des mots de passe SSH en clair dans Ansible (déconseillé en production).*

### Création de l'environnement virtuel
Positionnez-vous dans le répertoire du projet et initialisez l'environnement :
```bash
# 1. Accéder au répertoire du projet
cd ~/src/ansible-infra

# 2. Créer l'environnement virtuel
python3 -m venv .venv

# 3. Activer l'environnement virtuel
source .venv/bin/activate

# 4. Mettre à jour pip et installer Ansible
pip install --upgrade pip
pip install ansible

# 5. (Optionnel) Complétion automatique dans le terminal
pip install argcomplete
activate-global-python-argcomplete
```

---

## 2. Configuration SSH & Sudo sur Bazzite

Bazzite est un système d'exploitation immuable basé sur Fedora Atomic. L'accès SSH et la configuration sudo y possèdent quelques spécificités.

### Étape 1 : Activer le service SSH sur Bazzite
Sur Bazzite, le démon SSH (`sshd`) n'est pas actif par défaut. Vous devez l'activer physiquement sur la machine cible (via un terminal ou une console locale) :
```bash
sudo systemctl enable --now sshd
```

### Étape 2 : Générer et déployer la clé SSH depuis macOS
Il est recommandé d'utiliser une clé SSH dédiée de type **Ed25519** pour Ansible.

1. **Générer la clé** sur votre Mac :
   ```bash
   ssh-keygen -t ed25519 -C "ansible-control-mac" -f ~/.ssh/id_ed25519_ansible
   ```
   *(Appuyez sur Entrée pour ne pas mettre de passphrase, ou spécifiez-en une et configurez ssh-agent).*

2. **Copier la clé** sur Bazzite (remplacez `famille` par votre utilisateur Bazzite et `192.168.1.106` par son IP) :
   ```bash
   ssh-copy-id -i ~/.ssh/id_ed25519_ansible.pub famille@192.168.1.106
   ```

3. **Tester la connexion manuelle** pour valider la clé :
   ```bash
   ssh -i ~/.ssh/id_ed25519_ansible famille@192.168.1.106
   ```
   Vous devriez être connecté directement sans demande de mot de passe.

---

## 3. Dépannage (Troubleshooting)

### Problème 1 : Erreur "Connection refused" en SSH
* **Cause** : Le service SSH n'est pas démarré sur Bazzite, ou un pare-feu bloque le port 22.
* **Résolution** :
  1. Sur Bazzite, exécutez `sudo systemctl status sshd` pour valider qu'il est en cours d'exécution (*running*).
  2. Vérifiez le pare-feu local (Firewalld) sur Bazzite : `sudo firewall-cmd --state`. Si actif, vérifiez que le service SSH est autorisé : `sudo firewall-cmd --list-services`.

### Problème 2 : Demande de mot de passe lors du `become` (sudo) dans Ansible
* **Symptôme** : L'exécution d'un playbook échoue avec l'erreur `Missing sudo password` ou reste bloquée.
* **Cause** : Ansible tente d'exécuter une commande en tant que `root` (grâce à `become: true`), mais l'utilisateur `famille` sur Bazzite requiert un mot de passe pour sudo.
* **Résolution** :
  1. **Ansible Vault** : Saisissez le mot de passe sudo de Bazzite dans le fichier de secrets chiffré (`vault.yml`) sous la variable `vault_ansible_become_password`.
  2. **Pipelining** : Assurez-vous que le pipelining est bien activé dans [ansible.cfg](../ansible.cfg) :
     ```ini
     [defaults]
     pipelining = True
     ```
     Le pipelining permet d'exécuter plusieurs modules Ansible en une seule connexion SSH sans passer par des fichiers temporaires, évitant ainsi le besoin de TTY.
  3. **Become Flags** : Dans [ansible.cfg](../ansible.cfg), configurez les indicateurs de become :
     ```ini
     [privilege_escalation]
     become_flags = -S
     ```
     L'option `-S` force `sudo` à lire le mot de passe sur l'entrée standard (stdin) envoyée par Ansible, permettant l'authentification sans TTY interactif.

### Problème 3 : Erreurs de TTY (`sudo: a terminal is required to read the password`)
* **Symptôme** : Sudo refuse de s'exécuter car Ansible n'alloue pas de pseudo-terminal (TTY).
* **Cause** : La configuration SSH ou sudo locale de la machine cible requiert un TTY pour saisir le mot de passe.
* **Résolution** :
  1. Assurez-vous que l'option `ansible_ssh_extra_args='-o RequestTTY=no'` est présente dans [inventory.ini](../inventory.ini) pour forcer le comportement non-TTY qui est compatible avec le mode `-S` de sudo.
  2. Sur Bazzite, vérifiez avec `sudo visudo` qu'il n'y a pas de directive `Defaults requiretty` active. Sur Fedora moderne, cette directive est désactivée par défaut, mais elle peut bloquer Ansible si elle est présente.
