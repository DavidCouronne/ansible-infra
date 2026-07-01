# Audit de Sécurité et Améliorations Proposées

Ce document présente une analyse technique de la configuration actuelle des playbooks et templates du dépôt, identifie les risques de sécurité et propose des améliorations concrètes pour optimiser l'infrastructure.

---

## 1. Améliorations de la Sécurité

### A. Conflit Réseau et Exposition de Port (Critique)
Dans le template **[jellyfin.container.j2](../playbooks/templates/jellyfin.container.j2)**, on trouve ces lignes :
```ini
Network=host
PublishPort=127.0.0.1:8096:8096
```
* **Risque** : Sous Podman/Docker, si le mode réseau est défini sur `host` (`Network=host`), la directive `PublishPort` est **totalement ignorée**. Le conteneur écoute directement sur toutes les interfaces réseau physiques de l'hôte (`0.0.0.0:8096`). Si l'intention était de restreindre l'accès à `127.0.0.1` (par exemple pour passer par un reverse proxy comme Caddy/Nginx), cette restriction est actuellement inefficace.
* **Correction recommandée** :
  * Si Jellyfin doit être exposé uniquement sur `127.0.0.1` : Retirer `Network=host` pour laisser Podman créer son réseau privé virtuel (bridge) et router le trafic via `PublishPort=127.0.0.1:8096:8096`.
  * Si le mode `Network=host` est conservé (nécessaire pour la découverte de réseaux locaux comme le DLNA) : Retirer la ligne inutile `PublishPort` et sécuriser l'accès via le pare-feu système (Firewalld) de Bazzite.

### B. Désactivation de Seccomp (`Security=seccomp=unconfined`)
Le conteneur Jellyfin est configuré avec :
```ini
Security=seccomp=unconfined
```
* **Risque** : Cela désactive les filtres d'appels système (seccomp), augmentant considérablement la surface d'attaque sur le noyau linux de l'hôte en cas de compromission du conteneur. Cette option est souvent activée à tort pour résoudre des problèmes d'accélération matérielle graphique (transcodage GPU).
* **Correction recommandée** : Supprimer cette option. Si l'accélération matérielle est requise (Intel QuickSync ou VA-API), il est préférable d'exposer proprement les périphériques graphiques dans le Quadlet via :
  ```ini
  Device=/dev/dri/renderD128:/dev/dri/renderD128
  ```
  Et de garder les filtres de sécurité seccomp actifs.

### C. Emplacement des Identifiants SMB
Le playbook de montage dépose le fichier d'identifiants Samba dans `/var/home/{{ ansible_user }}/.smb/credentials`.
* **Risque** : Bien que les droits d'accès soient restreints à `0600`, placer des secrets système à l'intérieur d'un dossier utilisateur augmente le risque de fuite ou de suppression accidentelle par l'utilisateur. De plus, le montage systemd s'exécute avec les privilèges `root`.
* **Correction recommandée** : Centraliser les identifiants système dans `/etc/` :
  ```bash
  /etc/cifs-credentials/nas-video.cred
  ```
  Le dossier et le fichier doivent être la propriété exclusive de `root:root` avec les droits `0600`.

### D. Montage en Lecture Seule des Médiathèques
Actuellement, les volumes de médias sont montés en lecture seule (`:ro`) dans le conteneur Jellyfin :
```ini
Volume=/var/home/{{ ansible_user }}/nas/video:/media/video:ro
```
* **Bonne pratique validée** : C'est une excellente mesure de sécurité. Jellyfin n'a pas besoin d'écrire dans les répertoires de films/séries. Cela évite qu'une faille applicative ou une mauvaise manipulation n'altère ou ne supprime vos fichiers sur le NAS.

---

## 2. Optimisations et Améliorations Fonctionnelles

### A. Élimination de la Duplication de Code (Refactoring) (Appliqué)
* **État** : **Implémenté avec succès** ✅.
* **Réalisation** : Les playbooks dupliqués `mount_nas_video.yml` et `mount_nas_plex.yml` ont été supprimés et remplacés par le rôle local générique **[roles/nas_mount/](../roles/nas_mount/)**.
* **Fonctionnement** : La configuration des partages a été déplacée dans le fichier global **[group_vars/all/vars.yml](../group_vars/all/vars.yml)** sous la variable `nas_mounts` :
  ```yaml
  nas_mounts:
    - share: "video"
      subdir: "nas/video"
    - share: "PlexMediaServer"
      subdir: "nas/PlexMediaServer"
  ```
  Le playbook unifié **[playbooks/mount_nas.yml](../playbooks/mount_nas.yml)** boucle désormais sur cette liste et appelle le rôle générique. Cela réduit drastiquement la duplication de code et simplifie l'ajout de futurs points de montage.

### B. Utilisation du Contexte SELinux Directement dans le Point de Montage
Actuellement, le playbook SELinux configure le booléen `httpd_use_cifs` globalement.
* **Alternative plus propre** : On peut spécifier l'option `context` lors du montage Samba CIFS pour que les fichiers montés héritent directement du contexte SELinux nécessaire aux conteneurs, évitant ainsi d'activer des booléens globaux :
  Dans le template `cifs_mount.j2` :
  ```ini
  Options=credentials=...,context="system_u:object_r:container_file_t:s0"
  ```

### C. Remplacement des Chemins d'Accès Hardcodés (Appliqué)
* **État** : **Implémenté avec succès** ✅.
* **Réalisation** : Toutes les références reconstruites manuellement comme `/var/home/{{ ansible_user }}/` ont été remplacées par la variable dynamique Ansible standard **`{{ ansible_user_dir }}`** dans les playbooks, les variables de rôle, ainsi que dans le template de conteneur.
* **Bénéfice** : Les chemins s'adaptent désormais automatiquement selon l'utilisateur SSH actif et l'emplacement réel de son dossier personnel sur le système cible, améliorant grandement la portabilité du projet vers d'autres environnements.
