# ansible-infra

Infrastructure Ansible pour piloter et configurer une machine **Bazzite** (Fedora Atomic) depuis un Mac.

Ce dépôt regroupe les outils d'automatisation pour le déploiement de services de partage multimédia (Jellyfin, montages SMB) et de configuration du système cible.

---

## 📚 Guides et Documentation Détaillée

Pour faciliter la maintenance et l'évolution de cette infrastructure, la documentation a été découpée par thématiques :

1. 💻 **[Configuration Initiale & Dépannage](docs/setup_and_troubleshooting.md)** : Installation des prérequis sur macOS, configuration des accès SSH & Sudo sur Bazzite, et résolution des erreurs courantes (TTY, mot de passe requis).
2. 🔑 **[Gestion d'Ansible Vault](docs/vault_management.md)** : Fonctionnement de la clé de déchiffrement `.vault_pass` et administration des secrets de configuration.
3. 🚀 **[Guide de Déploiement Bazzite](docs/deployment_guide.md)** : Description de l'architecture des playbooks et ordre chronologique requis pour un déploiement sur une installation fraîche.
4. 🔒 **[Audit de Sécurité & Améliorations](docs/security_and_improvements.md)** : Analyse des points de sécurité critiques (Seccomp, Network/Port de Jellyfin) et pistes d'optimisation (refactoring de code).
5. 🛡️ **[Évaluation des Variables Sensibles](docs/sensitive_variables_evaluation.md)** : Analyse de la sensibilité des variables de l'infrastructure et guide pour migrer les adresses IP (comme `nas_ip`) et d'autres variables vers Ansible Vault.

---

## 🏛️ Architecture du Projet

```text
ansible-infra/
├── .venv/                          # Environnement virtuel Python (ignoré par git)
├── .vault_pass                     # Clé de déchiffrement Vault (ignoré par git)
├── ansible.cfg                     # Configuration globale Ansible
├── inventory.ini                   # Inventaire des machines hôtes
├── docs/                           # Guides et documentations techniques
├── group_vars/
│   └── all/
│       ├── vars.yml                # Variables publiques
│       └── vault.yml               # Secrets chiffrés (Ansible Vault)
├── roles/
│   └── nas_mount/                  # Rôle générique pour monter les partages NAS
│       ├── tasks/
│       │   └── main.yml            # Tâches de déploiement des montages
│       ├── vars/
│       │   └── main.yml            # Variables calculées dynamiquement
│       └── templates/
│           ├── cifs_mount.j2           # Template unité systemd .mount (CIFS)
│           ├── wait-for-nas-script.j2  # Script shell d'attente réseau
│           └── wait-for-nas.j2         # Service systemd d'attente réseau
├── playbooks/
│   ├── bazzite_config_selinux.yml  # Configuration globale SELinux
│   ├── mount_nas.yml               # Montage des partages NAS (fait appel au rôle)
│   ├── jellyfin.yml                # Déploiement Jellyfin (Podman Quadlet)
│   ├── bazzite_config_test.yml     # Diagnostics automatisés (Podman+Samba+SELinux)
│   └── templates/
│       └── jellyfin.container.j2   # Template Podman Quadlet Jellyfin
```

---

## 🖥️ Machines Gérées

| Alias | Adresse IP | Système d'Exploitation | Utilisateur Ansible |
|---|---|---|---|
| **bazzite-salon** | `192.168.1.106` | Bazzite (Fedora Atomic) | `famille` |

---

## ⚡ Commandes Rapides de Référence

### Tester la Connectivité
```bash
# Vérifier la liaison SSH simple (sans élévation become)
ansible bazzite_hosts -m ping -e "ansible_become=false"

# Vérifier l'escalade de privilèges root (become sudo)
ansible bazzite_hosts -m command -a "whoami" -b
```

### Administrer les Secrets (Vault)
```bash
# Consulter les secrets déchiffrés dans le terminal
ansible-vault view group_vars/all/vault.yml

# Modifier interactivement le coffre de secrets
ansible-vault edit group_vars/all/vault.yml
```

### Lancer un Playbook spécifique
```bash
# Exemple : Lancer le diagnostic système
ansible-playbook playbooks/bazzite_config_test.yml
```

---

## 📝 Contexte pour l'Assistant IA (Prompt)

Pour toute demande d'assistance future, vous pouvez copier le prompt de contexte suivant :

```text
J'ai un projet Ansible sur Mac pour piloter une machine Bazzite (Fedora Atomic) sur mon réseau local.
Voici le dépôt : https://github.com/DavidCouronne/ansible-infra

Stack : macOS + Ansible dans un venv Python, machine cible Bazzite (IP 192.168.1.106, user `famille`), NAS Synology (IP 192.168.1.8).

Points clés déjà configurés :
- Clé SSH dédiée : ~/.ssh/id_ed25519_ansible
- Option become non-interactif via `pipelining = True` et `become_flags = -S` dans ansible.cfg
- Montages NAS gérés via des unités systemd.mount générées dynamiquement
- Déploiement Jellyfin avec Podman Quadlet (.container utilisateur)
- Secrets centralisés dans Ansible Vault (clé locale non-commitée .vault_pass)

Je souhaite continuer à [DÉCRIRE LE BESOIN].
```