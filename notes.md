# Notes de référence du lab

## Environnement (10/11/2026)

| Outil | Version |
|---|---|
| kubectl (client) | v1.37.2 |
| kind | v0.33.0 |
| Kubernetes (node kind) | v1.37.0 |
| Docker | 29.1.3 |
| Terraform | 1.16.2 |
| Crossplane (cible) | v2.4.3 |

Machine : Ryzen 7 250, 16 threads, 29 Go de RAM, Ubuntu 25.10.

---

# Phase 1 : les bases de Kubernetes

## 1. Le cluster

Un cluster = un control plane + des nodes. Avec kind, chaque node est un conteneur Docker.

| Composant | Où | Rôle |
|---|---|---|
| kube-apiserver | control plane | Point d'entrée unique (API REST). kubectl, controllers et nodes passent tous par lui. |
| etcd | control plane | Base clé-valeur qui stocke l'état désiré de tous les objets. |
| kube-scheduler | control plane | Choisit le node de chaque pod. |
| kube-controller-manager | control plane | Fait tourner les boucles de réconciliation de base (ReplicaSet, Deployment...). |
| kubelet | chaque node | Démarre réellement les conteneurs et remonte leur état. |
| kube-proxy | chaque node | Traduit les Services en règles réseau (DNAT IP du Service vers IP des pods). |
| coredns | `kube-system` | Résout les noms des Services (`web.lab-dev.svc.cluster.local`). |
| kindnet | `kube-system` | CNI : attribue les IP des pods et route le trafic. |
| local-path-provisioner | `local-path-storage` | Crée un PersistentVolume quand un PVC apparaît. |

- Les composants du control plane tournent dans `kube-system`. On n'y déploie jamais ses applications.
- Ce sont des static pods : le kubelet les lance à partir de fichiers sur le node, sans passer par l'apiserver.
- L'apiserver est exposé sur `127.0.0.1:<port>` (port 6443 dans le conteneur). `kubectl cluster-info` le montre.
- kubectl ne fait rien lui-même : il envoie des requêtes à l'apiserver, configuré via `~/.kube/config`.

## 2. Objets, spec et status

- Tout est un objet décrit en YAML : `apiVersion`, `kind`, `metadata`, `spec`, `status`.
- `spec` = état désiré, écrit par l'utilisateur. `status` = état réel, écrit par le controller.
- Le controller compare l'état réel à la `spec` stockée dans etcd, pas au fichier. Le fichier n'est lu qu'au moment de l'`apply`.
- Les labels et selectors relient les objets entre eux (un Service trouve ses pods par leurs labels, pas par leur nom).

## 3. La réconciliation

Un controller observe l'état réel, le compare à l'état désiré, agit, met à jour le `status`, et recommence en boucle.

Exemple observé : suppression d'un pod d'un Deployment à 2 réplicas.
1. Le controller des ReplicaSets (dans kube-controller-manager) voit 1 pod au lieu de 2 et en crée un (`SuccessfulCreate`).
2. Le scheduler le place sur un node.
3. Le kubelet le démarre.

Le tout en moins d'une seconde, sans action humaine. Le pod est remplacé, pas redémarré : nouveau nom, nouvelle IP. C'est pour cela qu'on passe toujours par un Service.

Kubernetes réconcilie déjà seul ses propres objets. Crossplane étend ce mécanisme à des ressources extérieures au cluster.

## 4. Workloads et Service

Chaîne : Deployment → ReplicaSet → Pods. Le Service trouve les pods via son selector et publie leurs IP dans une EndpointSlice.

- Le suffixe du ReplicaSet (`web-6d89d48dd7`) est une empreinte du template de pod. Changer l'image crée un nouveau ReplicaSet.
- `port` : port sur lequel le Service écoute (les clients appellent `web:80`).
- `targetPort` : port du conteneur vers lequel le trafic est redirigé. Les deux peuvent différer (80 vers 8080).
- Plages d'adresses distinctes : Services en `10.96.x.x`, pods en `10.244.x.x`. L'IP d'un Service n'existe que dans les règles de kube-proxy.
- Un selector qui ne trouve aucun pod n'est pas une erreur : le Service est créé, l'EndpointSlice reste vide, aucun avertissement.

## 5. Mise à jour progressive (rolling update)

- Par défaut, le Deployment crée au plus 1 pod en plus (`maxSurge`) et n'en retire aucun tant que le nouveau n'est pas Ready (`maxUnavailable` arrondi à 0 pour 2 réplicas).
- Avec une image cassée : un seul nouveau pod en erreur, les anciens continuent de tourner, le site reste en ligne. La mise à jour est bloquée, sans coupure.
- Kubernetes ne fait pas de rollback automatique. Il se bloque proprement et attend.
- `BackOff` : le kubelet réessaie en espaçant les tentatives. Lui non plus ne renonce jamais.

## 6. Fichier contre cluster

- `kubectl rollout undo` remet la version précédente dans le cluster, mais ne touche pas au manifeste. Le prochain `apply` réintroduit la panne.
- Règle : on répare d'abord le service (undo si urgence), puis la source de vérité (le fichier versionné), puis `apply`.
- `kubectl diff -f fichier` montre l'écart entre fichier et cluster sans rien modifier. C'est l'équivalent le plus proche d'un `terraform plan`.
- Un `apply` qui réussit ne prouve pas que l'application fonctionne : `apply` rend la main dès que la `spec` est enregistrée. Il faut vérifier avec `rollout status`, les pods et un test réel.
- Lire la sortie de `apply` : `unchanged` = fichier et cluster d'accord. `configured` = quelque chose a changé. `created` = nouvel objet.

## 7. Dry-run serveur

- `kubectl apply -f fichier --dry-run=server` : l'apiserver valide chaque objet contre son schéma et contre l'état réel, mais n'enregistre rien.
- Limite : sur un cluster neuf, il échoue (`namespaces "lab-dev" not found`) si le fichier crée un namespace et y place des objets, car le namespace est validé mais jamais créé.
- Solutions pour la CI : créer les namespaces d'abord, les séparer dans un fichier, ou valider côté client avec kubeconform.

## 8. Diagnostic

### Ordre des commandes

```bash
kubectl get <type> -n <ns>                     # vue d'ensemble : READY, STATUS
kubectl describe <type> <nom> -n <ns>          # conditions et Events en bas
kubectl logs <pod> -n <ns> [--previous]        # sortie du programme (--previous : avant le dernier crash)
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl get pods -n <ns> -w                    # suivre les changements en direct
```

### Service qui ne répond pas

| Symptôme | Cause | Vérification |
|---|---|---|
| `bad address` | Le nom n'existe pas : faute de frappe, Service absent, mauvais namespace | `kubectl get svc -n <ns>` |
| `connection refused` ou timeout | Le nom existe, personne derrière : selector qui ne correspond pas, ou mauvais `targetPort` | `kubectl get endpointslices -n <ns>` puis `kubectl get pods -n <ns> --show-labels` |

### Pod qui ne démarre pas

| Statut | Cause |
|---|---|
| `Pending` | Aucun node ne peut l'accueillir (ressources, PVC non satisfait) |
| `ErrImagePull` puis `ImagePullBackOff` | Image introuvable. Lire le message : `NotFound` (tag absent), `unauthorized` (identifiants, imagePullSecret), `timeout` (réseau ou DNS vers le registre) |
| `CrashLoopBackOff` | Le conteneur démarre puis meurt en boucle : lire `logs --previous` |
| `OOMKilled` | Limite mémoire dépassée |

Un nom d'image court (`nginx`) est complété en `docker.io/library/nginx`.

### Tester depuis l'intérieur du cluster

```bash
kubectl run test --rm -it --image=busybox:1.37 -n <ns> --restart=Never -- wget -qO- -T 5 http://<service>
```

### Outils divers

- `kubectl explain <type>.<champ>` : documentation d'un champ, depuis le cluster.
- `kubectl api-resources` : tous les types et leurs abréviations (`deploy`, `svc`, `rs`...).

### Shell

`command not found` après une installation : le fichier existe-t-il (`ls`) ? son dossier est-il dans le PATH (`which`, `echo $PATH`) ?

---

## Pistes pour le benchmark

| Critère | Ce que la phase 1 a montré |
|---|---|
| Dérive | Kubernetes recrée seul un pod supprimé. Terraform ne réagit qu'au prochain `plan`. |
| État | Après un `rollout undo`, fichier et cluster divergent. Question à trancher : qui fait foi ? |
| Dégradation | Une mise à jour cassée se bloque sans coupure et sans rollback automatique. |
| Observabilité | Les erreurs de câblage (selector) sont silencieuses : objet valide, application muette. |
| Prévisualisation | `kubectl diff` existe, mais le dry-run serveur ne gère pas les dépendances internes à un fichier. `terraform plan` raisonne sur un graphe. |

## Mesures

| Date | Situation | RAM conteneur kind | Méthode |
|---|---|---|---|
| 10/11/2026 | Cluster vide, kind v0.33, 1 node | 579 Mo | `docker stats --no-stream` |
| 11/11/2026 | web (2 pods nginx) + postgres (1 pod), 1 node | 1,19 Go | `docker stats --no-stream` |

---

# Phase 2 : base persistante, réseau et droits

## 1. StatefulSet

Pourquoi PostgreSQL dans un StatefulSet et pas un Deployment :
- **Identité stable** : le pod s'appelle `postgres-0` et garde ce nom après recréation (un Deployment génère des noms aléatoires).
- **Un disque par pod, attaché au nom** : `volumeClaimTemplates` crée un PVC `<template>-<pod>`, ici `data-postgres-0`. Le nouveau `postgres-0` récupère le même PVC, donc les mêmes données.
- **Ordre** : démarrage et arrêt dans l'ordre (0, 1, 2...), utile pour un primaire et des réplicas.
- Avec un Deployment et un seul PVC `ReadWriteOnce`, la mise à jour progressive crée le nouveau pod avant de supprimer l'ancien : les deux réclament le même disque.

Un StatefulSet exige un **Service headless** (`clusterIP: None`) : pas d'IP virtuelle, coredns renvoie directement l'IP du pod, et chaque pod a un nom DNS propre (`postgres-0.postgres`).

Vérifié : table créée, pod supprimé, pod recréé, `SELECT` renvoie toujours la ligne.

## 2. Stockage

Chaîne : PVC (la demande) → StorageClass `standard` → local-path-provisioner → PV (le volume réel, un dossier sur le disque du node).

- Toujours vérifier où l'image écrit ses données avant de choisir le `mountPath` :
  `docker inspect <image> --format '{{json .Config.Env}}'` (PGDATA) et `--format '{{json .Config.Volumes}}'`.
- PostgreSQL 18 : `PGDATA=/var/lib/postgresql/18/docker`, on monte `/var/lib/postgresql`. Monter l'ancien chemin `/var/lib/postgresql/data` donnerait une base qui perd ses données au redémarrage, sans aucune erreur.
- `RECLAIM POLICY: Delete` : supprimer le PVC supprime le PV et les données.
- Supprimer un StatefulSet **ne supprime pas** ses PVC (prudence de Kubernetes).
- Supprimer le **namespace** supprime tout ce qu'il contient, PVC compris, donc les données (reclaim Delete). Recréer le namespace ne les ramène pas.
- Équivalent Crossplane : `deletionPolicy: Delete` ou `Orphan` sur une ressource managée.

## 3. Secret

- Un Secret est encodé en base64, **pas chiffré** : `kubectl get secret <nom> -o jsonpath='{.data.<clé>}' | base64 -d` le lit en clair.
- Ce qui le protège réellement : le RBAC (qui a le droit `get secrets` dans le namespace).
- Le pod le lit via `env.valueFrom.secretKeyRef` (nom du Secret + clé).
- Le mot de passe est en clair dans `postgres.yml` versionné : acceptable pour un lab, à ne jamais faire en vrai. Même problème que les secrets dans un `tfstate`.

## 4. NetworkPolicy

- `podSelector` = les pods **protégés**. `ingress.from.podSelector` = les pods **autorisés** à entrer.
- Par défaut le réseau est **ouvert** : tous les pods se parlent. Dès qu'une NetworkPolicy sélectionne un pod, tout ce qui n'est pas autorisé est bloqué pour lui (liste blanche).
- L'apiserver accepte toujours une NetworkPolicy, même si le CNI ne sait pas l'appliquer. Seul un test le prouve.
- **Vérifié avec kindnet** (kind v0.33) : intrus accepté sans la règle, bloqué avec. Les NetworkPolicy sont appliquées.
- Le trafic bloqué est **jeté** (DROP) : le client attend jusqu'au délai (`no response`).

## 5. ServiceAccount et RBAC

| Objet | Rôle |
|---|---|
| ServiceAccount | Identité d'un programme (`system:serviceaccount:<ns>:<nom>`) |
| Role / ClusterRole | Liste de permissions verbe + ressource, dans un namespace / dans tout le cluster |
| RoleBinding / ClusterRoleBinding | Attribue un rôle à une identité |

- Le RBAC est **toujours fermé par défaut** : sans binding, aucun droit, même en lecture.
- À l'inverse, le réseau est ouvert par défaut tant qu'aucune NetworkPolicy ne sélectionne le pod.
- `apiGroups: [""]` = groupe de base (pods, services, secrets, configmaps).
- Vérifié : compte `lecteur` sans RBAC `no no no no`, avec le rôle `lire-pods` `yes no no no` (list pods oui ; delete pods, get secrets, autre namespace non).
- Piège : `--as=$SA` avec une variable vide teste ta propre identité (admin) et répond `yes` partout. Toujours vérifier l'identité testée.

## 6. Diagnostic, ajouts de la phase 2

| Symptôme | Première hypothèse | Vérification |
|---|---|---|
| Délai dépassé (`no response`, timeout) vers un pod `Running` | NetworkPolicy qui bloque, ou labels du client qui ne correspondent pas au `from` | `kubectl get networkpolicy -n <ns>`, `kubectl describe networkpolicy <nom> -n <ns>`, `kubectl get pod <client> -n <ns> --show-labels` |
| Refus immédiat (`connection refused`) | Service sans endpoint ou mauvais port | `kubectl get endpointslices -n <ns>` |
| `<ressource> is forbidden` | Le ServiceAccount n'a pas le droit | `kubectl auth can-i <verbe> <ressource> -n <ns> --as=system:serviceaccount:<ns>:<sa>` |
| Données perdues après redémarrage | Mauvais `mountPath` | `kubectl exec <pod> -- sh -c 'echo $PGDATA'` comparé au `mountPath` |

Retrouver le ServiceAccount d'un pod : `kubectl get pod <pod> -n <ns> -o jsonpath='{.spec.serviceAccountName}'`.

Commandes utiles : `kubectl exec -it <pod> -- <cmd>` (commande dans le conteneur), `kubectl wait --for=condition=Ready pod/<pod>` (attendre une condition), `kubectl run ... --labels=app=x` (pod de test avec un label choisi).

## Pistes pour le benchmark (phase 2)

| Critère | Ce que la phase 2 a montré |
|---|---|
| Suppression | Supprimer un namespace emporte les données (PVC en reclaim Delete). Même risque que supprimer une XR ou un Claim avec `deletionPolicy: Delete`. |
| Sécurité de l'état | Secrets lisibles par quiconque a `get secrets`. Le tfstate pose le même problème côté Terraform. |
| Observabilité | NetworkPolicy et RBAC acceptés sans validation de leur effet réel : il faut des tests actifs. |
| Abstraction | Une seule variante dev = 4 fichiers et une dizaine d'objets. Ce sont eux que la Composition et le module Terraform devront générer. |
