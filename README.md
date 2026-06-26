# ansible-infra

Infrastructure Ansible pour piloter une machine **Bazzite** (Fedora Atomic) depuis un Mac.

## Architecture

```
ansible-infra/
├── .venv/                          # Environnement virtuel Python (ignoré par git)
├── .vault_pass                     # Clé de déchiffrement Vault (ignoré par git)
├── ansible.cfg                     # Configuration globale
├── inventory.ini                   # Déclaration des hôtes
├── group_vars/
│   └── all/
│       ├── vars.yml                # Variables publiques
│       └── vault.yml               # Secrets chiffrés (Ansible Vault)
├── playbooks/
│   ├── mount_nas_video.yml         # Montage SMB NAS → ~/nas/video
│   └── jellyfin.yml                # Déploiement Jellyfin (Podman quadlet)
└── templates/
    ├── systemd_mount.j2            # Template unité systemd .mount
    └── jellyfin.container.j2       # Template Podman quadlet Jellyfin
```

## Machines gérées

| Alias | IP | OS | Utilisateur |
|---|---|---|---|
| bazzite-salon | 192.168.1.106 | Bazzite (Fedora Atomic) | famille |

## Prérequis sur le Mac

```bash
# Activer l'environnement virtuel
cd ~/src/ansible-infra
source .venv/bin/activate

# Vérifier la connexion SSH
ssh -i ~/.ssh/id_ed25519_ansible famille@192.168.1.106
```

## Commandes Ansible

### Tester la connectivité

```bash
# Ping sans sudo
ansible bazzite_hosts -m ping -e "ansible_become=false"

# Vérifier sudo
ansible bazzite_hosts -m command -a "whoami" -b
```

### Montages NAS (SMB/CIFS via systemd)

```bash
# Vérifier en mode dry-run
ansible-playbook playbooks/mount_nas_video.yml --check

# Appliquer
ansible-playbook playbooks/mount_nas_video.yml

# Vérifier le montage sur Bazzite
ansible bazzite_hosts -m command -a "findmnt /var/home/famille/nas/video" -e "ansible_become=false"
```

### Jellyfin

```bash
# Vérifier en mode dry-run
ansible-playbook playbooks/jellyfin.yml --check

# Déployer
ansible-playbook playbooks/jellyfin.yml

# Vérifier le service
ansible bazzite_hosts -m command -a "systemctl --user status jellyfin.service" -e "ansible_become=false"
```

### Vault

```bash
# Voir les secrets
ansible-vault view group_vars/all/vault.yml

# Modifier les secrets
ansible-vault edit group_vars/all/vault.yml
```

## Points techniques notables

**Bazzite / Fedora Atomic**
- `sshd` n'est pas actif par défaut → `sudo systemctl enable --now sshd`
- `/home` est un lien symbolique vers `/var/home` → utiliser `/var/home/famille/` dans les unités systemd
- Pas de modification de `/etc/fstab` → on utilise des unités `systemd.mount` dans `/etc/systemd/system/`

**Ansible sur macOS**
- `EDITOR=nano` requis pour `ansible-vault create/edit` (ou `code --wait` pour VS Code)
- `pipelining = True` dans `ansible.cfg` nécessaire pour que `become` fonctionne sans TTY
- `become_flags = -S` pour passer le mot de passe sudo via stdin

**SELinux**
- Les volumes Podman nécessitent le contexte `container_file_t`
- Les montages NAS (CIFS) reçoivent ce contexte via l'option `context=` dans l'unité systemd
- Les disques internes btrfs/LUKS nécessitent `semanage fcontext` + `restorecon`

## Prompt de contexte pour Claude

> J'ai un projet Ansible sur Mac pour piloter une machine Bazzite (Fedora Atomic) sur mon réseau local.
> Voici le dépôt : https://github.com/DavidCouronne/ansible-infra
>
> Stack : macOS + Ansible dans un venv Python, machine cible Bazzite (IP 192.168.1.106, user `famille`), NAS Synology (IP 192.168.1.8).
>
> Points de configuration importants déjà en place :
> - Clé SSH `~/.ssh/id_ed25519_ansible`
> - `pipelining = True` et `become_flags = -S` dans ansible.cfg (nécessaire pour sudo sans TTY sur Bazzite)
> - Montages NAS via unités systemd.mount (pas de fstab, OS immuable)
> - Jellyfin déployé via Podman quadlet (.container)
> - Secrets dans Ansible Vault, clé dans `.vault_pass`
>
> Je veux continuer à [DÉCRIRE LA SUITE].