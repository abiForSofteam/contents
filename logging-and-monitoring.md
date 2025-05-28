Voici la première partie de votre cours détaillé sur le **Logging et le Monitoring dans Kubernetes**, axée sur la notion de **Labels et Selectors**, conformément aux exigences du niveau CKA.

---

# 🎯 Introduction : L'importance du Logging et du Monitoring dans Kubernetes

Dans un environnement Kubernetes, le **logging** et le **monitoring** sont essentiels pour :

* **Diagnostiquer les problèmes** : comprendre les défaillances des applications ou de l'infrastructure.
* **Assurer la performance** : surveiller l'utilisation des ressources et optimiser les performances.
* **Garantir la sécurité** : détecter les comportements anormaux ou les intrusions.
* **Maintenir la conformité** : répondre aux exigences réglementaires en matière de traçabilité.

Les **labels** et **selectors** jouent un rôle crucial dans ces processus en permettant une organisation efficace des ressources et une sélection précise des objets à surveiller ou à analyser.

---

# 🏷️ Labels et Selectors

## 📌 Définition

* **Labels** : Paires clé/valeur attachées aux objets Kubernetes (pods, services, etc.) pour les identifier de manière significative.
* **Selectors** : Mécanismes permettant de sélectionner un ensemble d'objets en fonction de leurs labels.([CKA Journey][1])

Ces outils sont fondamentaux pour organiser, sélectionner et gérer les ressources au sein d'un cluster Kubernetes.&#x20;

---

## 🧩 Scénarios et Résolutions

### 🔹 Scénario 1 : Organisation des logs par environnement

**Problématique** : Vous souhaitez séparer les logs des environnements de développement, de test et de production.

**Solution** :

1. **Ajouter un label d'environnement aux pods** :

   ```bash
   kubectl label pods <nom-du-pod> environment=production
   ```



2. **Utiliser un selector pour filtrer les logs** :

   ```bash
   kubectl logs -l environment=production
   ```



### 🔹 Scénario 2 : Surveillance spécifique d'une application

**Problématique** : Vous devez surveiller uniquement les pods d'une application spécifique.

**Solution** :

1. **Assigner un label d'application** :

   ```bash
   kubectl label pods <nom-du-pod> app=frontend
   ```



2. **Utiliser un selector pour surveiller ces pods** :

   ```bash
   kubectl get pods -l app=frontend
   ```



### 🔹 Scénario 3 : Déploiement ciblé avec labels

**Problématique** : Vous souhaitez déployer un service uniquement sur des pods avec un certain label.([Kubernetes][2])

**Solution** :

1. **Définir le label sur les pods** :

   ```bash
   kubectl label pods <nom-du-pod> tier=backend
   ```



2. **Configurer le service avec un selector correspondant** :

   ```yaml
   selector:
     tier: backend
   ```



### 🔹 Scénario 4 : Nettoyage des ressources obsolètes

**Problématique** : Vous devez identifier et supprimer les ressources obsolètes.

**Solution** :

1. **Lister les ressources avec un label spécifique** :

   ```bash
   kubectl get pods -l status=obsolete
   ```



2. **Supprimer ces ressources** :

   ```bash
   kubectl delete pods -l status=obsolete
   ```



### 🔹 Scénario 5 : Monitoring ciblé avec Prometheus

**Problématique** : Configurer Prometheus pour surveiller uniquement certains pods.

**Solution** :

1. **Assigner un label de monitoring aux pods** :

   ```bash
   kubectl label pods <nom-du-pod> monitoring=enabled
   ```



2. **Configurer Prometheus pour cibler ces pods via le label**.

---

Ces scénarios illustrent l'utilisation des labels et selectors pour une gestion efficace du logging et du monitoring dans Kubernetes.

---

# 🧪 **Taints and Tolerations**

## 🎯 Objectif pédagogique

Les **taints** et **tolerations** permettent de **contrôler la planification (scheduling)** des pods sur les nœuds d’un cluster Kubernetes. Dans le cadre du **logging** et du **monitoring**, cette capacité est cruciale pour :

* Diriger les workloads sensibles (comme ceux de la télémétrie, Prometheus, Fluentd) vers des nœuds dédiés.
* Isoler les charges système pour limiter la pollution des logs.
* Prioriser les nœuds en fonction de leur criticité ou performance.
* Empêcher certains pods d’être programmés sur des nœuds spécifiques.

---

## 📘 Rappels de syntaxe

* **Taint** (appliqué au nœud) :

  ```bash
  kubectl taint nodes <node-name> key=value:taint-effect
  ```

  *Effets possibles :* `NoSchedule`, `PreferNoSchedule`, `NoExecute`

* **Toleration** (défini dans la spec du pod) :

  ```yaml
  tolerations:
    - key: "key"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"
  ```

---

## 📚 5 scénarios concrets + résolutions

### 🔹 Scénario 1 : Dédié aux pods de monitoring

**Problématique** : Vous souhaitez réserver certains nœuds aux seuls outils de monitoring (Prometheus, Grafana…).

**Solution** :

1. Tainter les nœuds dédiés :

   ```bash
   kubectl taint nodes node-monitor monitoring=only:NoSchedule
   ```

2. Tolérer ce taint dans les pods Prometheus :

   ```yaml
   tolerations:
     - key: "monitoring"
       operator: "Equal"
       value: "only"
       effect: "NoSchedule"
   ```

---

### 🔹 Scénario 2 : Empêcher les pods applicatifs de se déployer sur des nœuds critiques

**Problématique** : Vous avez des nœuds critiques pour le système (logs, metrics) et vous ne voulez **aucun pod applicatif** dessus.

**Solution** :

1. Appliquer un taint bloquant :

   ```bash
   kubectl taint nodes node-logger reserved=true:NoSchedule
   ```

2. Aucun pod applicatif ne tolérera ce taint ⇒ ils seront exclus automatiquement.

---

### 🔹 Scénario 3 : Résilience des logs après une panne (NoExecute)

**Problématique** : Un nœud hébergeant Fluent Bit redémarre. Vous voulez que ses pods soient reschedulés ailleurs **uniquement s'ils tolèrent la condition.**

**Solution** :

1. Taint dynamique par Kubelet :
   (Exemple : `node.kubernetes.io/unreachable:NoExecute` appliqué automatiquement.)

2. Pods de logs doivent avoir :

   ```yaml
   tolerations:
     - key: "node.kubernetes.io/unreachable"
       operator: "Exists"
       effect: "NoExecute"
       tolerationSeconds: 60
   ```

---

### 🔹 Scénario 4 : Préférence douce avec `PreferNoSchedule`

**Problématique** : Vous voulez **éviter** que des pods normaux s’exécutent sur un nœud où vous collectez les logs, mais sans interdire strictement.

**Solution** :

```bash
kubectl taint nodes node-log logcollect=true:PreferNoSchedule
```

→ Les pods éviteront ce nœud sauf si aucun autre n’est disponible.

---

### 🔹 Scénario 5 : Identification des nœuds problématiques par les taints

**Problématique** : Un nœud est dégradé (latence élevée des métriques). Vous voulez forcer les pods à le quitter.

**Solution** :

1. Appliquer un taint temporaire :

   ```bash
   kubectl taint nodes node5 unstable=true:NoExecute
   ```

2. Seuls les pods avec :

   ```yaml
   tolerations:
     - key: "unstable"
       operator: "Equal"
       value: "true"
       effect: "NoExecute"
   ```

   resteront. Les autres seront expulsés.

---

## ✅ Résumé des bonnes pratiques

* **Séparer** les nœuds de logs/metrics via des taints dédiés.
* **Éviter** les interférences entre services d’observabilité et applications.
* **Tolérer intelligemment** les interruptions sur les pods critiques (logs/monitoring).
* **Utiliser `NoExecute`** pour garantir l’évacuation rapide en cas de défaillance.

---



# 🧩 **Node Selectors**

---

## 🎯 Objectif pédagogique

Les **Node Selectors** sont le moyen le plus simple de contrôler sur quel nœud un pod peut être programmé. Contrairement aux taints/tolerations (qui excluent ou incluent par contrainte), les Node Selectors **imposent** une correspondance directe entre un **label de nœud** et un **sélecteur du pod**.

Dans le contexte du **logging** et du **monitoring**, cette fonctionnalité est essentielle pour :

* Forcer les pods de télémétrie à résider uniquement sur des nœuds dédiés.
* Garantir que les outils sensibles ne cohabitent pas avec des workloads utilisateurs.
* Organiser le cluster par **rôle fonctionnel**.
* Simplifier le déploiement de collecteurs de logs ou d’agents metrics.

---

## 📘 Syntaxe de base

1. Label sur le nœud :

   ```bash
   kubectl label nodes <node-name> role=monitoring
   ```

2. Dans la spec du pod :

   ```yaml
   nodeSelector:
     role: monitoring
   ```

---

## 📚 5 scénarios concrets + résolutions

### 🔹 Scénario 1 : Réserver un nœud aux pods de Prometheus

**Problématique** : Vous voulez que **Prometheus** s'exécute uniquement sur un nœud haute capacité `node-metrics`.

**Solution** :

1. Ajouter un label au nœud :

   ```bash
   kubectl label nodes node-metrics monitoring=true
   ```

2. Définir le `nodeSelector` dans le `deployment.yaml` :

   ```yaml
   nodeSelector:
     monitoring: "true"
   ```

---

### 🔹 Scénario 2 : Isoler les logs des utilisateurs

**Problématique** : Fluent Bit collecte les logs des utilisateurs, mais ne doit pas être sur leurs nœuds.

**Solution** :

1. Label uniquement les nœuds système :

   ```bash
   kubectl label nodes node-sys role=logging
   ```

2. Dans le `DaemonSet` Fluent Bit :

   ```yaml
   nodeSelector:
     role: logging
   ```

---

### 🔹 Scénario 3 : Dédier un pod Grafana à un nœud GPU sans exécution GPU

**Problématique** : Un nœud dispose de beaucoup de ressources mais est inutilisé ; vous souhaitez y mettre Grafana.

**Solution** :

1. Label du nœud :

   ```bash
   kubectl label nodes gpu-node metrics-heavy=true
   ```

2. Spécification dans Grafana :

   ```yaml
   nodeSelector:
     metrics-heavy: "true"
   ```

> ⚠️ Le pod n’accède pas au GPU tant que `resources.limits` ne le demande pas.

---

### 🔹 Scénario 4 : Pod de log uniquement sur nœud localisé (par datacenter)

**Problématique** : Vous souhaitez que les pods de log restent **dans un datacenter spécifique** (ex : `dc=europe`).

**Solution** :

1. Label tous les nœuds européens :

   ```bash
   kubectl label nodes node-eu1 dc=europe
   ```

2. `nodeSelector` :

   ```yaml
   nodeSelector:
     dc: europe
   ```

---

### 🔹 Scénario 5 : Défaillance si aucun nœud ne correspond

**Problématique** : Que se passe-t-il si aucun nœud n’a le label ?

**Explication** : Le pod **ne sera jamais programmé**. Kubernetes ne génère pas d'erreur immédiate, mais le pod restera en **`Pending`**.

**Résolution** :

1. Vérifier les nœuds :

   ```bash
   kubectl get nodes --show-labels
   ```

2. Ajouter le bon label ou changer le `nodeSelector`.

---

## ✅ Résumé des bonnes pratiques

* Privilégier **Node Selectors** pour des affectations strictes simples.
* Toujours **documenter** les labels appliqués aux nœuds (`role`, `zone`, `env`...).
* Coupler avec les **taints/tolerations** pour plus de contrôle.
* Utiliser les Node Selectors pour vos **workloads de monitoring/logging dédiés** afin de garantir leur stabilité.

---



---

# 🧲 **Node Affinity** — CKA Niveau Avancé

---

## 🎯 Objectif pédagogique

**Node Affinity** est une extension avancée de `nodeSelector`. Elle permet une **expression plus riche et plus flexible** des contraintes de placement des pods sur les nœuds. Là où `nodeSelector` exige une correspondance stricte, **Node Affinity permet de spécifier des préférences et des contraintes plus granulaires**.

Dans le contexte du **logging** et du **monitoring**, cela est fondamental pour :

* Prioriser certains nœuds sans bloquer les autres.
* Imposer des règles **soft ou hard** pour le placement des agents de logs/metrics.
* Éviter les conflits de ressources sans sacrifier la disponibilité.
* Permettre la haute résilience du système de monitoring.
* Adapter le placement selon des contextes géographiques, de capacité, ou de criticité.

---

## 📘 Types d’affinité

| Type                                              | Sens                                                                                           |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `requiredDuringSchedulingIgnoredDuringExecution`  | Hard affinity. Le pod **doit** être programmé sur un nœud correspondant.                       |
| `preferredDuringSchedulingIgnoredDuringExecution` | Soft affinity. Le pod **devrait préférentiellement** être programmé sur un nœud correspondant. |

---

## 📚 5 scénarios concrets + résolutions

### 🔹 Scénario 1 : Prioriser les nœuds haute capacité pour Prometheus

**Problématique** : Prometheus devrait préférablement tourner sur des nœuds marqués `capacity=high`.

**Solution** :

1. Label des nœuds :

   ```bash
   kubectl label nodes node-a capacity=high
   ```

2. Dans le `Deployment` Prometheus :

   ```yaml
   affinity:
     nodeAffinity:
       preferredDuringSchedulingIgnoredDuringExecution:
         - weight: 100
           preference:
             matchExpressions:
               - key: capacity
                 operator: In
                 values:
                   - high
   ```

> ✅ Résultat : Prometheus sera **préférentiellement** planifié sur ces nœuds, mais pas bloqué si indisponibles.

---

### 🔹 Scénario 2 : Forcer Loki à tourner sur des nœuds spécifiques

**Problématique** : Loki doit obligatoirement tourner sur des nœuds labellisés `role=logging`.

**Solution** :

1. Label :

   ```bash
   kubectl label nodes node-log1 role=logging
   ```

2. Définition dans la spec :

   ```yaml
   affinity:
     nodeAffinity:
       requiredDuringSchedulingIgnoredDuringExecution:
         nodeSelectorTerms:
           - matchExpressions:
               - key: role
                 operator: In
                 values:
                   - logging
   ```

> ❗ Si aucun nœud ne correspond, le pod restera en `Pending`.

---

### 🔹 Scénario 3 : Déployer un collecteur de logs **dans le même datacenter**

**Problématique** : Vous voulez que les pods de logs restent dans le **même datacenter que d'autres pods** marqués `dc=europe`.

**Solution** :

1. Label :

   ```bash
   kubectl label nodes node-eu1 dc=europe
   ```

2. Affinité :

   ```yaml
   affinity:
     nodeAffinity:
       requiredDuringSchedulingIgnoredDuringExecution:
         nodeSelectorTerms:
           - matchExpressions:
               - key: dc
                 operator: In
                 values:
                   - europe
   ```

---

### 🔹 Scénario 4 : Minimiser les logs sur les nœuds de calcul

**Problématique** : Vous préférez ne pas utiliser les nœuds de calcul (`node-type=compute`) pour héberger Fluentd.

**Solution** : Exclure via un anti-affinity inversée.

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
            - key: node-type
              operator: NotIn
              values:
                - compute
```

> 🧠 Astuce : Cela **n’interdit pas** totalement, mais évite par préférence.

---

### 🔹 Scénario 5 : Multi-zones avec fallback

**Problématique** : Vous voulez que vos outils de monitoring tournent en priorité sur `zone=eu-west-1`, mais puissent basculer ailleurs.

**Solution** :

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
            - key: zone
              operator: In
              values:
                - eu-west-1
```

> 🔄 En cas de saturation ou de maintenance dans la zone cible, le pod est redirigé automatiquement.

---

## ✅ Résumé des bonnes pratiques

* Utilisez `preferredDuringScheduling...` pour la **résilience** et la **tolérance**.
* Réservez `requiredDuringScheduling...` aux cas critiques.
* Combinez `nodeSelector`, `taints`, et `affinity` pour une orchestration fine.
* Pensez **“résilience d’infrastructure monitoring”** : vos pods doivent pouvoir se relocaliser si nécessaire.



---

# 🧩 **DaemonSets** – CKA Niveau Avancé

---

## 🎯 Objectif pédagogique

Un **DaemonSet** permet de s'assurer que **chaque nœud (ou un sous-ensemble ciblé)** exécute une instance unique d'un pod. C'est un mécanisme central dans la mise en œuvre d’agents de **monitoring** (comme Node Exporter, cAdvisor) ou de **logging** (comme Fluentd, Fluent Bit, Logstash).

**Rôles clés dans le logging/monitoring** :

* Déploiement uniforme d’agents de collecte de logs/metrics.
* Garantie de couverture complète de l’infrastructure.
* Adaptation automatique aux ajouts/retraits de nœuds.
* Réduction du bruit opérationnel (ex. : éviter les pods orphelins ou manquants).

---

## 🧪 Scénarios & Résolutions (5 cas pratiques)

---

### 🔹 Scénario 1 : Déployer Fluent Bit sur tous les nœuds du cluster

**Problématique** : Assurer une **collecte uniforme des logs système** sur tous les nœuds.

**Solution** : Création d’un DaemonSet classique.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      name: fluent-bit
  template:
    metadata:
      labels:
        name: fluent-bit
    spec:
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:latest
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

```bash
kubectl apply -f fluentbit-daemonset.yaml
```

> ✅ Une instance par nœud, logs centralisés vers un backend (ex : Elasticsearch).

---

### 🔹 Scénario 2 : Déployer Node Exporter uniquement sur les nœuds Linux

**Problématique** : Certains nœuds exécutent Windows. Il faut **limiter le déploiement aux nœuds Linux**.

**Solution** : Utiliser `nodeSelector` dans le DaemonSet.

```yaml
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: linux
```

> ✅ Empêche les erreurs sur les nœuds incompatibles.

---

### 🔹 Scénario 3 : Empêcher le DaemonSet de tourner sur les nœuds masters

**Problématique** : On ne veut pas surcharger les nœuds masters avec des pods de logging.

**Solution** : Taint les nœuds masters et utilisez `tolerations`.

1. Tainter les nœuds masters (si ce n’est pas déjà fait) :

   ```bash
   kubectl taint nodes node-master-1 node-role.kubernetes.io/master=:NoSchedule
   ```

2. Ne pas ajouter de toleration dans le DaemonSet → pods non planifiés sur ces nœuds.

> ✅ Isolation des workloads de monitoring des composants critiques.

---

### 🔹 Scénario 4 : Mettre à jour Fluent Bit sans interruption

**Problématique** : Vous devez **mettre à jour l’image de Fluent Bit** sans perdre la collecte des logs.

**Solution** : Mise à jour par rolling update.

```yaml
updateStrategy:
  type: RollingUpdate
```

Puis modifier l’image :

```yaml
containers:
  - name: fluent-bit
    image: fluent/fluent-bit:2.2.0
```

> ✅ Kubernetes remplace les pods un par un.

---

### 🔹 Scénario 5 : Diagnostiquer une absence de pod Fluent Bit sur un nœud

**Problématique** : Un nœud ne possède pas le pod `fluent-bit`, alors qu’il devrait.

**Solution** :

1. Vérifier l’état du DaemonSet :

   ```bash
   kubectl get daemonset fluent-bit -n logging
   ```

2. Vérifier si le nœud est schedulable :

   ```bash
   kubectl describe node <nom-node>
   ```

3. Vérifier les taints :

   ```bash
   kubectl get nodes -o json | jq '.items[].spec.taints'
   ```

4. Vérifier les events :

   ```bash
   kubectl describe pod fluent-bit-<hash> -n logging
   ```

> 🎯 Identifier s’il s’agit d’un problème de `taint`, d’espace disque, ou de restriction de `nodeSelector`.

---

## ✅ Bonnes pratiques

* Ne jamais utiliser DaemonSet sans `tolerations` lorsque des `taints` sont en place.
* Toujours **monitorer l’état global** des pods DaemonSet (pour détecter des manques).
* Préférer `hostPath` pour la lecture des fichiers de logs locaux.
* Implémenter une **update strategy** pour les upgrades contrôlés.
* Utiliser un **namespace dédié** (`monitoring`, `logging`) pour l’isolation logique.

---



---

# 🧩 **Static Pods** – CKA Niveau Avancé

---

## 🎯 Intérêt du sujet

Les **Static Pods** sont des pods qui ne sont pas gérés par le plan de contrôle Kubernetes (c’est-à-dire, ils ne sont pas créés via l’API server) mais directement gérés par le **kubelet** sur un nœud spécifique. Ils sont souvent utilisés pour :

* Héberger des composants critiques (ex : etcd, kube-apiserver sur les nœuds masters).
* Déployer des agents de monitoring/logging **avant** que le cluster ne soit complètement fonctionnel.
* Diagnostiquer un nœud localement sans dépendance à l’API server.

### 🧠 Pourquoi c’est important pour le logging/monitoring ?

* Les static pods peuvent être utilisés pour assurer une **surveillance de bas niveau**, même quand l’API Server est inopérant.
* Ils permettent d’installer un agent de monitoring/logging **de secours** sur un nœud problématique.
* Les logs des static pods sont **en local**, dans `/var/log/pods` et peuvent fournir un historique utile en cas de panne grave.

---

## 🧪 Scénarios & Résolutions (5 cas pratiques)

---

### 🔹 Scénario 1 : Déployer un agent Node Exporter comme static pod sur un nœud

**Problème** : Le cluster n’est pas encore initialisé, mais vous souhaitez surveiller les performances d’un nœud.

**Solution** : Créer un fichier dans `/etc/kubernetes/manifests/`.

```yaml
# /etc/kubernetes/manifests/node-exporter.yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-exporter
  labels:
    app: monitoring
spec:
  containers:
    - name: node-exporter
      image: prom/node-exporter
      ports:
        - containerPort: 9100
      volumeMounts:
        - mountPath: /proc
          name: proc
          readOnly: true
  volumes:
    - name: proc
      hostPath:
        path: /proc
```

✅ Le kubelet détecte automatiquement le fichier YAML et crée le pod.

---

### 🔹 Scénario 2 : Obtenir les logs d’un static pod en cas de problème de kube-apiserver

**Problème** : Le kube-apiserver est inactif. Vous devez comprendre pourquoi.

**Solution** :

1. Lire les logs directement sur le nœud :

   ```bash
   journalctl -u kubelet
   ```

2. Accéder aux logs du pod static :

   ```bash
   cat /var/log/pods/kube-system_kube-apiserver-<node>_*/kube-apiserver/*.log
   ```

✅ Les logs étant gérés par le kubelet, ils sont disponibles même sans API Server.

---

### 🔹 Scénario 3 : Surveiller l’état d’un static pod sur un nœud particulier

**Problème** : Un static pod d’export métrique ne fonctionne pas correctement.

**Solution** :

```bash
# Voir les pods sur le nœud local
crictl pods

# Voir les logs du conteneur
crictl ps -a
crictl logs <container-id>
```

💡 `crictl` est souvent installé avec `containerd` ou `cri-o`.

---

### 🔹 Scénario 4 : Un fichier manifest erroné empêche le démarrage du static pod

**Problème** : Vous modifiez un fichier dans `/etc/kubernetes/manifests/`, mais le pod ne démarre pas.

**Solution** :

1. Vérifier les logs du kubelet :

   ```bash
   journalctl -xeu kubelet
   ```

2. Vérifier les erreurs de parsing YAML :

   * Kubelet **n’interprétera pas** le fichier s’il est invalide.
   * Corriger les erreurs d’indentation ou de syntaxe.

✅ Le kubelet est strict avec les fichiers statiques.

---

### 🔹 Scénario 5 : Vous voulez superviser la disponibilité des composants static pods du plan de contrôle

**Problème** : Vous devez collecter des métriques sur les pods `etcd`, `kube-scheduler`, etc. – tous déployés en static pods.

**Solution** :

1. Configurer Prometheus pour découvrir les endpoints via les fichiers `kubelet` exposés en `/metrics`.
2. Accéder aux metrics via le port 10255 (non sécurisé, à restreindre !) ou configurer le port 10250 (authentifié).
3. Exemple de `prometheus.yml` :

```yaml
- job_name: 'kubelets'
  scheme: https
  static_configs:
    - targets:
      - 192.168.1.100:10250
  tls_config:
    insecure_skip_verify: true
```

✅ Permet une visibilité fine sur les composants critiques.

---

## ✅ Bonnes pratiques

* Toujours valider les fichiers YAML via `yamllint` avant dépôt dans `/etc/kubernetes/manifests/`.
* Utiliser `crictl` ou `journalctl` pour déboguer localement.
* Ne pas déployer à la légère en static pod : impossible à gérer par `kubectl`.
* Préférer un DaemonSet si le cluster est stable et initialisé.
* Utiliser un processus CI/CD même pour les fichiers de static pods.

---



---

# 🧩 **Multiple Schedulers** – Niveau CKA

---

## 🎯 Intérêt du sujet

Kubernetes utilise un **scheduler** (planificateur) par défaut qui décide **sur quel nœud** déployer un pod, en fonction de différentes contraintes (ressources, affinités, etc.).
Cependant, dans certains cas avancés, il est utile, voire nécessaire, de **déployer un scheduler personnalisé**. C’est là que les **multiple schedulers** entrent en jeu.

### 🔎 Pourquoi c’est utile ?

* Pour des **workloads critiques** nécessitant une stratégie de planification personnalisée.
* Pour séparer le scheduling de **différents types d'applications** (par ex. batch vs temps réel).
* Pour des **tests ou expérimentations** sans impacter le scheduler par défaut.
* Pour observer et auditer les décisions de planification de manière fine.

### 🧠 Rôle dans le logging & monitoring :

* Tracer quel scheduler a pris une décision.
* Diagnostiquer des pods non planifiés.
* Superviser la santé et l’activité de plusieurs schedulers.

---

## 🧪 Scénarios & Résolutions (5 cas pratiques)

---

### 🔹 Scénario 1 : Déployer un scheduler personnalisé dans le cluster

**Problème** : Vous souhaitez qu’un scheduler planifie uniquement les pods avec une annotation spéciale.

**Solution** :

1. Créer un fichier de déploiement personnalisé du scheduler :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-scheduler
  namespace: kube-system
spec:
  containers:
    - name: scheduler
      image: k8s.gcr.io/kube-scheduler:v1.29.0
      command:
        - kube-scheduler
        - --scheduler-name=my-scheduler
        - --leader-elect=false
```

2. Le pod est géré par le kubelet comme static pod ou déploiement régulier.

✅ Votre scheduler est maintenant actif et prêt à gérer des pods.

---

### 🔹 Scénario 2 : Forcer un pod à être planifié par un scheduler personnalisé

**Problème** : Vous avez deux schedulers, et vous voulez qu’un pod spécifique soit pris en charge par le second.

**Solution** :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  schedulerName: my-scheduler
  containers:
    - name: busybox
      image: busybox
      command: ["sleep", "3600"]
```

✅ Le champ `schedulerName` indique clairement quel scheduler doit planifier ce pod.

---

### 🔹 Scénario 3 : Diagnostiquer un pod non planifié

**Problème** : Un pod n’est pas assigné à un nœud. Aucun événement n’est généré.

**Solution** :

1. Vérifier le champ `schedulerName` :

   ```bash
   kubectl get pod test-pod -o=jsonpath='{.spec.schedulerName}'
   ```

2. Vérifier si le scheduler indiqué est actif :

   ```bash
   kubectl get pods -n kube-system | grep my-scheduler
   ```

3. Afficher les logs du scheduler :

   ```bash
   kubectl logs my-scheduler -n kube-system
   ```

✅ Cela permet de savoir si le scheduler est tombé, indisponible ou inopérant.

---

### 🔹 Scénario 4 : Ajouter de la journalisation personnalisée à un scheduler

**Problème** : Vous voulez suivre toutes les décisions de planification.

**Solution** :

* Utiliser les options de log du scheduler :

```bash
--v=4
```

* Ou bien rediriger la sortie dans un fichier si vous gérez en static pod :

```yaml
command:
  - kube-scheduler
  - --scheduler-name=my-scheduler
  - --v=5
```

✅ Vous obtenez plus de détails sur le processus de planification.

---

### 🔹 Scénario 5 : Monitorer l’activité des schedulers multiples avec Prometheus

**Problème** : Vous devez superviser les performances et la latence des planificateurs.

**Solution** :

1. Exposer les métriques via l’option `--bind-address=0.0.0.0 --secure-port=10259`.

2. Ajouter dans Prometheus :

```yaml
- job_name: 'custom-schedulers'
  metrics_path: /metrics
  static_configs:
    - targets: ['my-scheduler.kube-system.svc.cluster.local:10259']
```

✅ Vous pouvez comparer les performances entre plusieurs planificateurs.

---

## ✅ Bonnes pratiques

* Utiliser un `schedulerName` explicite et cohérent dans vos pods.
* Activer une politique de logs adaptée à vos besoins (`--v=4` ou supérieur).
* Isoler le rôle de chaque scheduler via des labels ou annotations.
* Surveiller la charge et les décisions avec Prometheus ou les logs JSON du scheduler.
* Ne pas rendre un scheduler critique si celui-ci n’est pas HA ou supervisé.

---



---

# 🧩 **Configuring Scheduler Profiles** – Niveau CKA

---

## 🎯 Intérêt du sujet

Kubernetes permet une configuration fine du **scheduler** à l’aide de **profiles de scheduling**.
Les **Scheduler Profiles** permettent de définir **plusieurs logiques de planification** à l’intérieur d’un même scheduler. Chaque profile peut avoir ses propres plugins activés/désactivés et ses propres règles de priorisation.

### 🔎 Pourquoi est-ce important ?

* Pour **adapter dynamiquement le comportement du scheduler** selon des cas d’usage (batch, temps réel, criticité, etc.).
* Pour **ajouter ou désactiver des plugins** de scoring, de préemption ou de filtrage.
* Pour **gérer des workloads hétérogènes** sans créer plusieurs schedulers.
* Pour **centraliser la planification** tout en diversifiant la stratégie.

### 🧠 Rôle dans le Logging & Monitoring :

* Auditer quel profile a été utilisé.
* Analyser les logs pour comprendre pourquoi un pod a été ou non planifié.
* Surveiller le comportement d’un profile dans Prometheus.

---

## 🧪 Scénarios & Résolutions (5 cas pratiques)

---

### 🔹 Scénario 1 : Activer plusieurs profiles dans un seul scheduler

**Problème** : Vous voulez qu’un même scheduler utilise plusieurs stratégies de planification selon le `schedulerName`.

**Solution** : Configurer le scheduler avec un fichier YAML.

1. Créez un fichier de configuration `/etc/kubernetes/scheduler-config.yaml` :

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
    plugins:
      score:
        enabled:
          - name: NodeResourcesBalancedAllocation
  - schedulerName: realtime-scheduler
    plugins:
      score:
        enabled:
          - name: NodeResourcesLeastAllocated
```

2. Lancer le scheduler avec :

```bash
kube-scheduler --config=/etc/kubernetes/scheduler-config.yaml
```

✅ Vous avez maintenant deux comportements différents selon le schedulerName utilisé dans les pods.

---

### 🔹 Scénario 2 : Forcer un pod à utiliser un profile particulier

**Problème** : Vous souhaitez qu’un pod utilise le profile "realtime-scheduler".

**Solution** :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: realtime-app
spec:
  schedulerName: realtime-scheduler
  containers:
    - name: app
      image: busybox
      command: ["sleep", "3600"]
```

✅ Le pod sera traité selon la logique du profile correspondant dans le scheduler configuré.

---

### 🔹 Scénario 3 : Diagnostiquer un mauvais comportement de planification

**Problème** : Des pods sont mal répartis sur les nœuds malgré la configuration.

**Solution** :

1. Activer les logs verbeux du scheduler :

   ```bash
   kube-scheduler --v=5 --config=/etc/kubernetes/scheduler-config.yaml
   ```

2. Vérifier quel plugin est utilisé pour le scoring :

   ```bash
   grep "Score plugin" /var/log/kube-scheduler.log
   ```

✅ Permet de confirmer que le bon profile a été appliqué.

---

### 🔹 Scénario 4 : Ajouter un plugin personnalisé dans un profile

**Problème** : Vous avez un plugin personnalisé à insérer dans le processus de planification.

**Solution** :

1. Compiler le plugin avec Go, l’enregistrer dans le scheduler.

2. Ajouter dans la configuration :

```yaml
profiles:
  - schedulerName: custom-scheduler
    plugins:
      score:
        enabled:
          - name: MyCustomScorer
    pluginConfig:
      - name: MyCustomScorer
        args:
          weight: 10
```

✅ Le scheduler utilisera votre plugin avec le profile configuré.

---

### 🔹 Scénario 5 : Monitorer les profils de scheduling avec des métriques

**Problème** : Vous souhaitez observer le comportement de chaque profile.

**Solution** :

1. Activer l’endpoint de métriques (déjà activé par défaut sur `:10259`).

2. Interroger les métriques :

```bash
curl -k https://localhost:10259/metrics | grep scheduler_profile
```

3. Ajouter un job Prometheus :

```yaml
- job_name: 'scheduler-profiles'
  metrics_path: /metrics
  static_configs:
    - targets: ['localhost:10259']
```

✅ Vous obtenez des métriques par profile comme `scheduler_plugin_execution_duration_seconds`.

---

## ✅ Bonnes pratiques

* Toujours documenter les profils et leur usage prévu.
* Vérifier que les `schedulerName` utilisés dans les pods correspondent bien à ceux des profiles.
* Ne pas dupliquer inutilement des profils ; factoriser les comportements communs.
* Utiliser les logs de haut niveau (`--v=5`) pour observer les décisions prises.
* Superviser les performances de chaque profile avec Prometheus.

---

## 📘 Conclusion générale du module Logging & Monitoring (niveau CKA)

Les notions couvertes dans ce cours — de *Labels and Selectors* à *Scheduler Profiles* — permettent non seulement une meilleure maîtrise du **comportement du planificateur Kubernetes**, mais aussi une **visibilité opérationnelle** essentielle à tout ingénieur DevOps ou SRE.

Vous avez désormais :

* des **scénarios réels**, directement applicables,
* les **commandes Kubernetes** précises pour diagnostiquer et intervenir,
* une compréhension approfondie du rôle de **l’observabilité** dans la fiabilité d’un cluster.

---








