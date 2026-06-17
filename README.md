# D-ploiement-Alloy-avec-Ansible


# Procédure de déploiement de Grafana Alloy avec Ansible

## 1. Objectif

Cette procédure permet de déployer **Grafana Alloy** via Ansible sur une machine locale.

Le déploiement réalise les actions suivantes :

- téléchargement du paquet `.deb` de Grafana Alloy ;
- installation du paquet ;
- copie du fichier de configuration Alloy ;
- activation et démarrage du service `alloy`.

---

## 2. Arborescence attendue

L’arborescence du projet Ansible doit être similaire à celle-ci :

```bash
/etc/ansible/
├── deploy_grafana_alloy.yml
└── roles/
    └── grafana_alloy/
        ├── defaults/
        │   └── main.yml
        ├── files/
        │   └── config.alloy
        └── tasks/
            └── main.yml
```

---

## 3. Création du playbook

Créer le fichier suivant :

```bash
/etc/ansible/deploy_grafana_alloy.yml
```

Contenu du playbook :

```yaml
---
- name: Deploy grafana alloy
  hosts: localhost
  become: true

  roles:
    - grafana_alloy

  vars:
    version: "1.17.0"
    src: "/etc/ansible/roles/grafana_alloy/files/config.alloy"
    dest: "/etc/alloy/config.alloy"
```

> Remarque : le dossier standard Ansible est `files` avec un **s**.

---

## 4. Création du rôle Ansible

Créer le rôle Ansible :

```bash
mkdir -p /etc/ansible/roles/grafana_alloy/{defaults,files,tasks}
```

---

## 5. Création des variables par défaut

Créer le fichier :

```bash
/etc/ansible/roles/grafana_alloy/defaults/main.yml
```

Contenu :

```yaml
---
version: ""
src: ""
dest: ""
```

Ces variables permettent de définir :

| Variable | Description |
|---|---|
| `version` | Version de Grafana Alloy à installer |
| `src` | Chemin source du fichier de configuration Alloy |
| `dest` | Chemin de destination de la configuration Alloy |

---

## 6. Création du fichier de configuration Alloy

Créer le fichier :

```bash
/etc/ansible/roles/grafana_alloy/files/config.alloy
```

Contenu :

```hcl
logging {
  level = "warn"
}

prometheus.exporter.unix "node" {
}

prometheus.scrape "node_metrics" {
  targets          = prometheus.exporter.unix.node.targets
  scrape_interval = "15s"
  forward_to      = [prometheus.remote_write.mimir.receiver]
}

prometheus.remote_write "mimir" {
  endpoint {
    url = "http://192.168.91.140:9009/api/v1/push"

    headers = {
      "X-Scope-OrgID" = "demo",
    }
  }
}
```

Ce fichier permet à Grafana Alloy de :

- récupérer les métriques système via l’exporter Unix ;
- scraper les métriques toutes les 15 secondes ;
- envoyer les métriques vers Grafana Mimir via l’endpoint remote write.

---

## 7. Création des tâches du rôle

Créer le fichier :

```bash
/etc/ansible/roles/grafana_alloy/tasks/main.yml
```

Contenu :

```yaml
---
- name: Download grafana alloy
  ansible.builtin.get_url:
    url: "https://github.com/grafana/alloy/releases/download/v{{ version }}/alloy-{{ version }}-1.amd64.deb"
    dest: "/tmp/alloy-{{ version }}-1.amd64.deb"
    mode: '0644'

- name: Install grafana alloy
  ansible.builtin.apt:
    deb: "/tmp/alloy-{{ version }}-1.amd64.deb"

- name: Copy config
  ansible.builtin.copy:
    src: "{{ src }}"
    dest: "{{ dest }}"
    owner: root
    group: root
    mode: '0644'

- name: Enable, start and daemon-reload grafana alloy
  ansible.builtin.systemd:
    name: alloy
    enabled: true
    state: started
    daemon_reload: true
```

---

## 8. Vérification de la syntaxe Ansible

Avant d’exécuter le playbook, vérifier sa syntaxe :

```bash
ansible-playbook /etc/ansible/deploy_grafana_alloy.yml --syntax-check
```

Résultat attendu :

```bash
playbook: /etc/ansible/deploy_grafana_alloy.yml
```

---

## 9. Exécution du playbook

Lancer le déploiement :

```bash
ansible-playbook /etc/ansible/deploy_grafana_alloy.yml
```

---

## 10. Vérification du service Alloy

Vérifier que le service est actif :

```bash
systemctl status alloy
```

Le service doit apparaître en état :

```bash
active (running)
```

---

## 11. Vérification de la configuration copiée

Vérifier que le fichier de configuration est bien présent :

```bash
ls -l /etc/alloy/config.alloy
```

Afficher son contenu :

```bash
cat /etc/alloy/config.alloy
```

---

## 12. Redémarrage manuel du service si nécessaire

Après modification du fichier `config.alloy`, redémarrer Alloy :

```bash
systemctl restart alloy
```

Puis vérifier son statut :

```bash
systemctl status alloy
```

---

## 13. Points de contrôle

Vérifier les éléments suivants :

- le paquet Grafana Alloy est bien installé ;
- le fichier `/etc/alloy/config.alloy` existe ;
- le service `alloy` est activé au démarrage ;
- le service `alloy` est en cours d’exécution ;
- l’endpoint Mimir est joignable depuis la machine Alloy ;
- le port `9009` du load-balancer Mimir est accessible.

Test réseau possible :

```bash
curl -I http://192.168.91.140:9009
```

---

## 14. Commandes utiles

Afficher les logs du service Alloy :

```bash
journalctl -u alloy -f
```

Redémarrer le service :

```bash
systemctl restart alloy
```

Arrêter le service :

```bash
systemctl stop alloy
```

Activer le service au démarrage :

```bash
systemctl enable alloy
```

Désactiver le service au démarrage :

```bash
systemctl disable alloy
```

---

## 15. Résumé

Cette procédure permet de déployer automatiquement Grafana Alloy avec Ansible grâce à :

- un playbook principal ;
- un rôle dédié `grafana_alloy` ;
- un fichier de variables par défaut ;
- un fichier de configuration Alloy ;
- une liste de tâches Ansible pour télécharger, installer, configurer et démarrer Alloy.
