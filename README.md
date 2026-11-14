# Lab : Crossplane vs Terraform

Provisionner la même pile applicative deux fois, une fois avec Crossplane, une fois avec Terraform, sur le même cluster Kubernetes local, puis comparer les deux outils sur des critères fixés à l'avance.

**Statut : en cours.** Ce dépôt est mis à jour à chaque étape. L'état d'avancement ci-dessous dit ce qui est fait et ce qui ne l'est pas encore.

## La question

Pour orchestrer le déploiement de classes de services qui combinent des workloads existants et des services managés, dans quels cas Crossplane est-il préférable à Terraform, et inversement ?

Je ne cherche pas un verdict binaire. Le résultat attendu est un tableau et une conclusion du type « voici dans quel cas chacun gagne ».

## La démarche

1. Construire la pile à la main, pour comprendre ce que les outils devront générer.
2. Figer la grille de critères avant toute mesure : [benchmark.md](benchmark.md).
3. Construire la pile avec Crossplane, jouer les scénarios, mesurer.
4. Construire la même pile avec Terraform, jouer les mêmes scénarios, mesurer.
5. Comparer et conclure, en séparant ce qui a été mesuré de ce qui relève du jugement.

Toute modification de la grille après coup est tracée dans le fichier, avec sa date et sa raison.

## La pile

| Ressource | dev | prod |
|---|---|---|
| Réseau : namespace + NetworkPolicy | web vers postgres autorisé | idem + refus par défaut du reste |
| Serveur PostgreSQL (StatefulSet, volume persistant) | 1 instance, 1 Gi | 1 instance, 5 Gi, limites CPU/RAM |
| Service web (Deployment) | 1 réplica | 3 réplicas |
| Base applicative et compte SQL, créés par l'API de PostgreSQL | 1 base, 1 compte | 1 base, 1 compte avec limite de connexions |

La quatrième ressource n'est pas un objet Kubernetes : elle est créée en parlant à une API externe. Elle représente le cas d'un service managé.

## Environnement

Tout tourne en local, sans compte cloud.

| Outil | Version |
|---|---|
| kind | v0.33.0 |
| Kubernetes | v1.37.0 |
| kubectl | v1.37.2 |
| Helm | v4.3.1 |
| Crossplane | v2.4.3 |
| provider-sql | v0.16.2 |
| Terraform | v1.16.2 |

## État d'avancement

| Étape | État |
|---|---|
| Cluster kind, Deployment et Service écrits à la main, tests de panne | Fait |
| PostgreSQL persistant, NetworkPolicy, RBAC, chacun vérifié par un test | Fait |
| Grille du benchmark : 16 critères, protocoles, biais connus | Fait |
| Installation de Crossplane et de provider-sql, première ressource managée | Fait |
| Abstraction Crossplane : XRD et Composition, variantes dev et prod | À faire |
| Mesures côté Crossplane | À faire |
| La même pile en Terraform, mêmes mesures | À faire |
| Tableau comparatif et conclusion | À faire |
| Pipeline GitHub Actions qui valide les deux implémentations | À faire |

## Premiers constats

Ce ne sont pas encore les résultats du benchmark, seulement ce que la mise en place a montré.

- **Coût d'installation.** Le conteneur du cluster passe de 1,20 Go à 1,37 Go avec le cœur de Crossplane (21 CRD), puis à 1,50 Go avec provider-sql (60 CRD au total). Mesuré avec `docker stats`.
- **Un petit provider ajoute 39 CRD.** Chaque type de ressource a sa CRD, en double : le modèle historique, global au cluster, et le modèle de la v2, rattaché à un namespace.
- **Couverture des providers.** La pile prévoyait un bucket MinIO. Avant de l'installer, j'ai constaté que le dépôt MinIO était archivé et que le provider Crossplane correspondant n'avait pas eu de release depuis plus d'un an. Je l'ai remplacé par une base PostgreSQL gérée par provider-sql, qui est maintenu.
- **Une erreur mal signalée.** Ma NetworkPolicy bloquait le provider, qui vit dans un autre namespace que la base. Le statut de la ressource affichait encore une ancienne erreur DNS, et les journaux du provider étaient vides. Il a fallu un test de connexion manuel pour trouver la cause.
- **Limite du dry-run serveur.** `kubectl apply --dry-run=server` échoue sur un cluster neuf si le fichier crée un namespace et y place des objets. À traiter dans la future pipeline.

## Structure

```text
k8s-basics/     # la pile écrite à la main (référence)
crossplane/     # provider, configuration, ressources managées
benchmark.md    # grille de critères et protocoles de mesure
notes.md        # notes de référence, mesures brutes
```

## Reproduire

```bash
kind create cluster --name lab

# la pile de référence
kubectl apply -f k8s-basics/web.yml
kubectl apply -f k8s-basics/postgres.yml
kubectl apply -f k8s-basics/netpol.yml
kubectl apply -f k8s-basics/rbac.yml

# Crossplane
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace --version 2.4.3
kubectl wait --for=condition=Available deployment --all -n crossplane-system --timeout=180s

# provider et première ressource managée
kubectl apply -f crossplane/provider-sql.yml
kubectl wait --for=condition=Healthy provider/provider-sql --timeout=180s
kubectl apply -f crossplane/providerconfig.yml
kubectl apply -f crossplane/database.yml
kubectl get databases.postgresql.sql.m.crossplane.io -n lab-dev
```

## Limites

- Un cluster local à un seul node : les mesures montrent des mécanismes, pas un comportement à grande échelle.
- Pas d'API de cloud réelle : les ressources sont des objets Kubernetes et une base PostgreSQL.
- Les mots de passe présents dans les manifestes sont des valeurs de lab, sans aucune valeur ailleurs. Dans un contexte réel, ils ne seraient jamais versionnés.
