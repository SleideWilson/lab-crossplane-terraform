# Benchmark Crossplane vs Terraform

Grille figée avant toute mesure. Les critères ne changent plus une fois les mesures commencées.
Toute modification ultérieure est notée en bas, avec sa date et sa raison.

## Question

Pour orchestrer le déploiement de classes de services qui combinent des workloads existants et des services managés,
dans quels cas Crossplane est-il préférable à Terraform, et inversement ?

## Conditions communes

- Même cluster kind (v0.33.0, Kubernetes v1.37.0, 1 node), même machine.
- Même pile, mêmes variantes, mêmes valeurs.
- Mesures répétées 3 fois, on note la médiane.
- Chaque mesure : date, version des outils, commande exacte, résultat brut.

## Pile de référence

| Ressource | dev | prod |
|---|---|---|
| Réseau : namespace + NetworkPolicy | web vers postgres autorisé | idem + refus par défaut du reste |
| Serveur PostgreSQL | 1 instance, 1 Gi | 1 instance, 5 Gi, limites CPU/RAM |
| Service : web | 1 réplica | 3 réplicas |
| Base applicative : base + compte SQL, créés par l'API de PostgreSQL | 1 base, 1 compte | 1 base, 1 compte avec limite de connexions |

## Grille

Type : **M** = mesurable, **J** = jugement, **R** = raisonnement.
Priorité : **P** = critère prioritaire, mesuré avec le plus de soin (3 répétitions). Les autres sont traités plus vite.

Les numéros sont stables : ils ne suivent pas l'ordre des familles.

### A. L'outil peut-il faire le travail ?

| # | Critère | Type | Priorité | Scénario | Unité |
|---|---|---|---|---|---|
| 10 | Couverture et maturité des providers | J | | Providers utilisés dans le lab | indicateurs du protocole |
| 13 | Ordonnancement et dépendances | M | P | Créer une pile neuve dont les ressources dépendent les unes des autres | ordre de création, erreurs transitoires, secondes |
| 2 | Abstraction | M | P | Ajouter une variante `staging` | fichiers modifiés, lignes ajoutées |
| 14 | Évolution d'une classe | M | P | Modifier le modèle alors que dev et prod sont déployés | instances touchées, action humaine, interruption |

### B. Tient-il en exploitation ?

| # | Critère | Type | Priorité | Scénario | Unité |
|---|---|---|---|---|---|
| 1 | Dérive | M | P | Modifier à la main les réplicas de web, puis supprimer la base applicative | secondes, action humaine oui/non, extrait de log |
| 4 | Dégradation et reprise | M | P | La base applicative échoue (accès à PostgreSQL refusé), puis retour arrière | état des 3 autres ressources, étapes du retour |
| 6 | Suppression | M | | Supprimer la pile dev entière | ce qui disparaît, ce qui reste |
| 5 | Observabilité | J | | Pendant le scénario 4 | nombre de commandes pour trouver la cause |
| 7 | Prévisualisation | M | | Voir un changement avant de l'appliquer | oui/non, information affichée |

### C. Tient-il à plusieurs et à l'échelle ?

| # | Critère | Type | Priorité | Scénario | Unité |
|---|---|---|---|---|---|
| 3 | État, concurrence et droits | M + J | | Deux personnes appliquent un changement en même temps | conflit, écrasement ou verrou |
| 8 | Coût d'exécution | M | | Outil installé, pile au repos | Mo de RAM, nombre de CRD, secondes |
| 9 | Passage à 200 classes | R | | Extrapolation depuis 2, 3, 8 et 14 | argumentation |

### D. Peut-on y aller sans tout casser ?

| # | Critère | Type | Priorité | Scénario | Unité |
|---|---|---|---|---|---|
| 12 | Coexistence et migration | M | P | Terraform crée une ressource, Crossplane l'observe puis la reprend | recréée oui/non, interruption oui/non |
| 15 | Intégration | R | | Un système externe doit créer une instance et suivre son état | argumentation |

### E. Est-ce un pari raisonnable ?

| # | Critère | Type | Priorité | Scénario | Unité |
|---|---|---|---|---|---|
| 11 | Courbe d'apprentissage | J | | Mon parcours + cas d'une équipe Terraform | heures, notions nouvelles |
| 16 | Licence, gouvernance, pérennité | J | | Les deux outils et leurs providers | indicateurs du protocole |

## Protocoles

Les commandes Crossplane exactes seront confirmées à l'installation (phase 3). La méthode, elle, ne change pas.

### 1. Dérive

- **Geste** : `kubectl scale deploy/web --replicas=5` à la main. Puis, séparément, suppression de la base applicative avec `psql` (`DROP DATABASE`).
- **Observation** : chronomètre entre le geste et le retour à la valeur du code (`kubectl get deploy web -w`, liste des bases avec `psql -c '\l'`).
- **Des deux côtés** : même geste. Côté Terraform, attendre 10 minutes sans rien faire, puis noter ce que montre `terraform plan`.
- **Ce que je note** : délai de correction, action humaine nécessaire ou non, ligne de log ou d'événement qui signale la dérive.
- **Ce qui peut fausser** : l'intervalle de réconciliation de Crossplane est configurable. Je note la valeur utilisée (celle par défaut).

### 2. Abstraction

- **Geste** : commit propre, puis ajout de la variante `staging` (2 réplicas, 2 Gi), puis nouveau commit.
- **Mesure** : `git diff --numstat <avant> <après>` donne les fichiers touchés et les lignes ajoutées.
- **Deux chiffres séparés** : ce que modifie l'équipe plateforme (XRD + Composition, ou module) et ce qu'écrit l'équipe qui consomme (XR, ou appel du module).
- **Ce qui peut fausser** : le style d'écriture. Même convention des deux côtés : pas de commentaires comptés, une valeur par ligne.

### 3. État et concurrence

- **Geste** : deux terminaux, deux changements contradictoires lancés en même temps sur la même pile (réplicas à 2 dans l'un, à 4 dans l'autre).
- **Terraform** : deux `terraform apply -auto-approve` simultanés. Une fois avec l'état local, une fois avec un backend partagé et verrou.
- **Crossplane** : deux `kubectl apply` simultanés sur la même XR.
- **Ce que je note** : message affiché à chacun, valeur finale, et surtout si le perdant est prévenu que son changement a été refusé ou écrasé.
- **Jugement** : où vit l'état, qui peut le lire, ce qu'il contient de sensible (`tfstate` contre etcd).
- **Droits** : peut-on autoriser une équipe à créer une pile dans son namespace sans lui donner accès aux ressources sous-jacentes ni aux secrets ? Test avec un ServiceAccount restreint et `kubectl auth can-i`. Côté Terraform : de quels droits a besoin celui qui lance `apply` ?

### 4. Dégradation

- **Geste** : mettre un mauvais mot de passe dans les identifiants que l'outil utilise pour PostgreSQL, puis créer une pile neuve. La base applicative ne peut pas être créée.
- **Observation après 2 minutes** : état des 3 autres ressources (créées ou non, fonctionnelles ou non), état global affiché par l'outil.
- **Reprise** : remettre le bon mot de passe. Chronomètre jusqu'à la pile complète. Action humaine nécessaire ou non.
- **Retour arrière** : revenir au commit précédent et appliquer. Nombre d'étapes, état final.

### 5. Observabilité

- **Point de départ identique** : pendant le scénario 4, je sais seulement que « la pile n'est pas prête ».
- **Mesure** : nombre de commandes jusqu'au message qui nomme la cause (authentification refusée par PostgreSQL). Je note la liste des commandes.
- **Ce que je note aussi** : le message cite-t-il la cause en clair ? Peut-on suivre l'avancement pendant le déploiement ?
- **Limite** : c'est un jugement. Je connais le cluster, un nouveau venu mettrait plus de temps.

### 6. Suppression

- **Préparation** : une table avec une ligne dans la base applicative.
- **Geste** : supprimer la XR dev, ou `terraform destroy` sur dev.
- **Vérification** : `kubectl get all,pvc,networkpolicy,secret -n lab-dev`, `kubectl get pv`, liste des bases si le serveur existe encore.
- **Ce que je note** : durée, confirmation demandée ou non, ce qui reste, sort des données.
- **Second passage** : avec la protection de chaque outil (`deletionPolicy: Orphan`, `prevent_destroy`). Ce qui change.

### 7. Prévisualisation

- **Changement testé** : web de 1 à 2 réplicas, disque de 1 à 2 Gi. Rien n'est appliqué.
- **Terraform** : `terraform plan`.
- **Crossplane** : `kubectl diff` sur la XR, et le rendu local de la Composition (`crossplane render`, à confirmer).
- **Ce que je note** : voit-on l'effet sur les ressources finales, ou seulement sur l'objet de haut niveau ? Faut-il un cluster ? Le changement destructeur (disque) est-il signalé ?

### 8. Coût d'exécution

- **RAM** : `docker stats --no-stream lab-control-plane` à trois moments. Pile écrite à la main (référence : 1,19 Go), outil installé, pile au repos depuis 10 minutes.
- **CRD** : `kubectl get crds --no-headers | wc -l` avant et après l'installation.
- **Terraform** : rien ne tourne au repos. Je mesure la durée et le pic mémoire d'un apply avec `/usr/bin/time -v terraform apply`.
- **Aussi** : durée entre la commande et la pile prête, des deux côtés.

### 9. Passage à 200 classes (raisonnement)

À partir des mesures 2, 3, 8 et 14 : coût d'une classe supplémentaire, coût d'un changement commun à toutes les classes,
nombre de CRD et charge de l'apiserver, organisation des équipes. Présenté comme une extrapolation, pas comme un résultat.

### 10. Maturité des providers

- **Pour chaque provider utilisé, des deux côtés** : date de la dernière release, nombre de releases sur 12 mois, version (0.x ou 1.x), issues ouvertes, mainteneur (éditeur ou communauté), documentation avec exemples.
- **Sources** : GitHub, Upbound Marketplace, Terraform Registry. Date de consultation notée.
- **Plus** : chaque bug ou manque rencontré pendant le lab, noté au moment où il arrive.

### 11. Courbe d'apprentissage

- **Mesure** : temps passé par séance et par outil, noté dans `notes.md`. Liste des notions nouvelles par outil. Nombre de blocages qui ont demandé une aide extérieure.
- **Limite** : une seule personne, qui connaissait déjà Terraform. C'est un témoignage, pas une statistique.

### 12. Coexistence et migration

- **Préparation** : Terraform crée une base applicative. J'y crée une table avec une ligne et je note l'identifiant interne de la base (`SELECT oid FROM pg_database WHERE datname = '...'`).
- **Étape 1** : Crossplane le déclare en observation seule (`external-name` + `managementPolicies: ["Observe"]`). Vérifier que son état remonte dans `status.atProvider`.
- **Étape 2** : passage en gestion complète.
- **Étape 3** : retrait de la base de l'état Terraform sans le détruire (`terraform state rm`).
- **Preuve à chaque étape** : la ligne est toujours là, l'identifiant interne n'a pas changé. Donc la base n'a pas été recréée.
- **Ce que je note** : nombre d'étapes, risques à chaque étape (deux outils qui gèrent la même ressource en même temps).

### 13. Ordonnancement et dépendances

- **Dépendances de la pile** : web a besoin de la base prête et de son mot de passe. La base applicative a besoin du serveur PostgreSQL prêt et joignable. Tout a besoin du namespace.
- **Geste** : créer une pile neuve, sans rien préparer.
- **Mesure** : ordre réel de création (`kubectl get <types> -n <ns> --sort-by=.metadata.creationTimestamp`), nombre d'erreurs transitoires et de redémarrages de pods avant convergence, durée jusqu'à la pile prête.
- **Ce que je note** : comment la dépendance s'écrit dans le code (référence, `depends_on`, condition de disponibilité), et comment une valeur produite par une ressource (mot de passe, adresse) arrive à une autre.
- **Ce qui peut fausser** : les images déjà présentes sur le node. Je les précharge des deux côtés.

### 14. Évolution d'une classe

- **Situation de départ** : dev et prod déployés et fonctionnels.
- **Geste** : modifier le modèle côté plateforme (changer la version de l'image de web, ajouter une limite mémoire).
- **Mesure** : quelles instances changent, à quel moment, avec ou sans action de l'équipe qui consomme. Une boucle `wget` tourne pendant le changement pour détecter une interruption.
- **Déploiement progressif** : peut-on appliquer le changement à dev d'abord, à prod ensuite ? Avec quel mécanisme (version de module, révision de Composition, à confirmer) ?
- **Retour arrière** : étapes pour revenir à l'ancien modèle, sur une seule instance puis sur toutes.

### 15. Intégration (raisonnement)

Un orchestrateur existant doit créer une instance, suivre son avancement et lire son état final.
- Crossplane : appel à l'API Kubernetes, état lisible dans l'objet (`status`), notifications par watch.
- Terraform : ligne de commande à exécuter, état à stocker et verrouiller, ou service intermédiaire.
Je décris ce qu'il faudrait écrire dans chaque cas. Pas de mesure.

### 16. Licence, gouvernance, pérennité

- **Pour chaque outil** : licence actuelle, changements de licence passés, qui décide de la feuille de route, existence d'un fork, statut dans une fondation.
- **Sources** : dépôts officiels, annonces des éditeurs, site de la CNCF. Date de consultation notée.
- **Aucune affirmation de mémoire** : chaque fait est vérifié à la source au moment de la rédaction.

## Résultats

### Critère 2, abstraction : côté Crossplane (18/11/2026)

Crossplane 2.4.3, function-patch-and-transform v0.11.2. Variante ajoutée : `medium` (2 réplicas web).
Mesure : `git diff --numstat` entre les commits `eb3a8d7` et `7aa38e1`.

| Qui | Fichier | Lignes ajoutées | Lignes supprimées | Changement utile |
|---|---|---|---|---|
| Équipe plateforme | `xrd-appstack.yml` | 2 | 2 | `medium` ajouté à l'`enum` (l'autre ligne est la description) |
| Équipe plateforme | `composition-appstack.yml` | 1 | 0 | `medium: 2` dans la table du patch |
| Équipe applicative | `xr-shop-staging.yml` (nouveau) | 8 | 0 | la demande elle-même, `size: medium` |

- **Plateforme : 2 fichiers, 2 lignes utiles.** Les deux sont obligatoires : sans l'`enum`, l'apiserver refuse la demande ; sans la table, la demande n'est pas traduite.
- **Équipe applicative : 1 fichier de 8 lignes,** sans rien savoir des Deployments ni de PostgreSQL.
- **Résultat obtenu :** `shop-staging` READY, Deployment `2/2`, base `shop_staging` créée.
- **Effet sur l'existant :** les Deployments de dev et de prod n'ont pas été modifiés (`metadata.generation` toujours à 1). Changer la Composition a pourtant déclenché une réconciliation de toutes les AppStack : sans effet ici, puisque leur rendu ne change pas. À approfondir au critère 14.
- **Mesuré une fois.** Le résultat est déterministe (un diff), la répétition n'apporte rien.
- **À comparer** avec le même ajout côté Terraform (phase 6).

## Biais connus

- Crossplane est construit en premier, Terraform en second : le second profite de ce que j'ai appris sur le premier. Effet limité, car je pratiquais déjà Terraform.
- Je connais Terraform mieux que Crossplane : risque de juger Crossplane plus difficile par manque de pratique plutôt que par conception.
- Échelle d'un lab : les mesures montrent des mécanismes, pas un comportement à 30 000 VM.

## Hors périmètre

- L'outil maison d'OUTSCALE : pas d'accès. Section de raisonnement dans la conclusion, avec la même grille.
- Les API IaaS réelles (AWS, OpenStack, OUTSCALE) : remplacées par des ressources Kubernetes et par l'API SQL de PostgreSQL.
- Le multi-région et le multi-fournisseur : un cluster local à un node ne permet aucune conclusion honnête.

## Méthode de choix des critères

1. Ce qui distingue les deux outils par conception (ponctuel ou continu, fichier d'état ou cluster, module ou API).
2. Le cycle de vie d'une ressource : créer, modifier, tomber en panne, supprimer, migrer.
3. Le contexte de l'offre : ordonnancement, 200 classes de services, continuité de l'existant.
4. Relecture avec le modèle de qualité ISO/IEC 25010 pour repérer les oublis.

Dans un cadre réel, les critères et leur poids seraient fixés avec l'équipe qui décide.

## Modifications de la grille

| Date | Changement | Raison |
|---|---|---|
| 12/11/2026 | Quatrième ressource : le bucket MinIO est remplacé par une base applicative et son compte, créés dans PostgreSQL (provider-sql côté Crossplane, provider postgresql côté Terraform). Scénarios 1, 4, 6, 12 et 13 adaptés. | Vérification avant installation : le dépôt MinIO est archivé (dernière release en octobre 2025) et provider-minio n'a pas eu de release depuis juillet 2025. Aucune mesure n'avait été prise. Constat reporté au critère 10. |
| 11/11/2026 | Ajout des critères 13 à 16, des droits dans le critère 3, des familles A à E et des priorités | Relecture de la grille avant toute mesure : l'ordonnancement et l'évolution d'une classe, au cœur de l'offre, n'étaient pas testés |
