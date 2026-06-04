Hachim Kidjei MAHAMADENE — Velat TAS

# TP — Buildah, Trivy, Dive & Helm/Kubernetes — Projet Odoru

## 1. Contexte du rendu

Ce rendu correspond au TP **Buildah, Trivy, Dive & Helm/Kubernetes**.

Le sujet de référence mentionne l’application **MIAGE Bank**. Dans ce rendu, les mêmes objectifs techniques ont été appliqués au projet microservices **Odoru**, une application de gestion d’un club de danse rythmique.

Le projet Odoru présente une architecture comparable à celle attendue dans le sujet :

- frontend web ;
- API Gateway ;
- microservices métier ;
- bases PostgreSQL ;
- Config Server ;
- Discovery Service ;
- authentification centralisée avec Keycloak ;
- déploiement Kubernetes avec Helm ;
- analyse d’images avec Trivy et Dive ;
- approche GitOps avec ArgoCD.

L’objectif du rendu est de démontrer la capacité à :

- construire des images OCI avec Buildah ;
- analyser la sécurité des images avec Trivy ;
- analyser et optimiser les couches d’images avec Dive ;
- packager et déployer l’application avec Helm/Kubernetes ;
- sécuriser le déploiement Kubernetes ;
- automatiser la configuration Keycloak ;
- démontrer une approche GitOps avec ArgoCD.

---

## 2. Présentation synthétique de l’application Odoru

Odoru est une application microservices destinée à gérer un club de danse rythmique.

Elle permet notamment de gérer :

- les membres du club ;
- les cours ;
- les compétitions ;
- les badges ;
- les statistiques ;
- l’authentification des utilisateurs via Keycloak.

L’application est composée des éléments suivants :

| Élément | Rôle |
| --- | --- |
| `odoru-front` | Interface web de démonstration |
| `api-gateway` | Point d’entrée unique vers les microservices |
| `member-service` | Gestion des membres |
| `course-service` | Gestion des cours |
| `competition-service` | Gestion des compétitions |
| `badge-service` | Gestion des badges |
| `statistics-service` | Agrégation de statistiques |
| `config-server` | Configuration centralisée |
| `discovery-service` | Découverte de services avec Eureka |
| `keycloak` | Authentification et gestion des rôles |
| PostgreSQL | Bases de données des microservices |

---

## 3. Architecture technique

L’architecture déployée suit une logique microservices :

```text
Navigateur
   |
   | http://localhost:30081
   v
Frontend Odoru
   |
   | API calls
   v
API Gateway
   |
   +--> member-service        --> member-postgres
   +--> course-service        --> course-postgres
   +--> competition-service   --> competition-postgres
   +--> badge-service         --> badge-postgres
   +--> statistics-service
   |
   +--> Keycloak
   |
   +--> Discovery Service
   |
   +--> Config Server
```

En Kubernetes, l’application est déployée dans le namespace :

```text
odoru
```

Les services exposés localement sont :

| Service | Type | Port |
| --- | --- | --- |
| `front` | NodePort | `30081` |
| `api-gateway` | NodePort | `30080` |
| autres services | ClusterIP | ports internes |

---

## 4. Structure du dépôt

Structure principale du projet :

```text
.
├── build-reports
│   ├── dive
│   ├── oci
│   └── trivy
├── infrastructure
│   ├── argocd
│   │   └── odoru-application.yaml
│   ├── config-repo
│   └── helm
│       └── odoru
├── odoru-front
├── scripts
│   ├── analyze-dive.sh
│   ├── build-all.sh
│   ├── scan-trivy.sh
│   ├── seed-kubernetes-data.sh
│   └── setup-keycloak-odoru.sh
├── services
│   ├── api-gateway
│   ├── badge-service
│   ├── competition-service
│   ├── config-server
│   ├── course-service
│   ├── discovery-service
│   ├── member-service
│   └── statistics-service
└── tp-buildah-trivy-dive-helm
    └── hachim-mahamadene
        └── README.md
```

---

## 5. Prérequis techniques

L’environnement de test utilisé repose sur :

| Outil | Rôle |
| --- | --- |
| WSL2 Ubuntu | Environnement Linux |
| Docker Desktop | Runtime conteneur et Kubernetes local |
| Kubernetes `docker-desktop` | Cluster local |
| Buildah | Construction d’images OCI |
| Trivy | Analyse de vulnérabilités |
| Dive | Analyse des couches d’images |
| Helm | Packaging et déploiement Kubernetes |
| Java 17 | Compilation des microservices Spring Boot |
| Node.js / npm | Build du frontend |
| kubectl | Administration Kubernetes |
| ArgoCD | Déploiement GitOps |

Vérification de l’environnement :

```bash
kubectl config current-context
kubectl get nodes
helm version
buildah --version
trivy --version
dive --version
java -version
node -v
npm -v
```

Résultat attendu :

```text
docker-desktop
docker-desktop Ready
Helm OK
Buildah OK
Trivy OK
Dive OK
Java 17 OK
Node.js OK
npm OK
```

---

# Partie A — Buildah, Trivy et Dive

## 6. Objectifs de la Partie A

La Partie A du TP porte sur :

- l’analyse comparative Docker / Buildah ;
- la construction d’images OCI avec Buildah ;
- l’analyse de sécurité avec Trivy ;
- l’analyse des couches avec Dive ;
- l’automatisation de la chaîne de build.

---

## 7. Analyse comparative Docker / Buildah

Docker est une solution complète de gestion de conteneurs, reposant historiquement sur un démon centralisé.

Buildah est un outil spécialisé dans la construction d’images OCI. Il permet de construire des images sans dépendre d’un démon Docker permanent.

| Critère | Docker | Buildah |
| --- | --- | --- |
| Architecture | Client + démon Docker | Sans démon centralisé |
| Usage principal | Build, run, gestion complète | Build d’images OCI |
| Sécurité | Dépend du démon Docker | Réduction de la surface d’attaque |
| Format | Images Docker/OCI | Images OCI |
| Intérêt DevSecOps | Standard très répandu | Adapté aux pipelines rootless et OCI |

Dans ce rendu, Buildah a été utilisé pour construire les images des composants Odoru.

---

## 8. Images construites avec Buildah

Les images suivantes ont été construites :

```text
localhost/odoru/config-server:1.0.0
localhost/odoru/discovery-service:1.0.0
localhost/odoru/api-gateway:1.0.0
localhost/odoru/member-service:1.0.0
localhost/odoru/course-service:1.0.0
localhost/odoru/competition-service:1.0.0
localhost/odoru/badge-service:1.0.0
localhost/odoru/statistics-service:1.0.0
localhost/odoru/front:1.0.0
```

Chaque service dispose d’un `Containerfile`.

Exemple :

```text
services/member-service/Containerfile
services/api-gateway/Containerfile
odoru-front/Containerfile
```

---

## 9. Script de build intégré

Le script principal de build est :

```text
scripts/build-all.sh
```

Il permet de construire toutes les images Odoru avec Buildah.

Commande utilisée :

```bash
./scripts/build-all.sh 1.0.0
```

Vérification :

```bash
buildah images | grep odoru
```

Le script permet de centraliser la construction des images, de limiter les erreurs de manipulation et de rendre le processus reproductible.

---

## 10. Export des images OCI

Les images construites sont exportées sous forme d’archives OCI dans :

```text
build-reports/oci
```

Exemples :

```text
build-reports/oci/member-service-1.0.0.tar
build-reports/oci/api-gateway-1.0.0.tar
build-reports/oci/front-1.0.0.tar
```

Ces archives permettent ensuite de charger les images dans Docker Desktop pour le déploiement Kubernetes local :

```bash
docker load -i build-reports/oci/config-server-1.0.0.tar
docker load -i build-reports/oci/discovery-service-1.0.0.tar
docker load -i build-reports/oci/api-gateway-1.0.0.tar
docker load -i build-reports/oci/member-service-1.0.0.tar
docker load -i build-reports/oci/course-service-1.0.0.tar
docker load -i build-reports/oci/competition-service-1.0.0.tar
docker load -i build-reports/oci/badge-service-1.0.0.tar
docker load -i build-reports/oci/statistics-service-1.0.0.tar
docker load -i build-reports/oci/front-1.0.0.tar
```

Vérification :

```bash
docker images | grep odoru
```

---

## 11. Analyse Trivy

Trivy a été utilisé pour analyser les vulnérabilités des images.

Script utilisé :

```text
scripts/scan-trivy.sh
```

Commande :

```bash
./scripts/scan-trivy.sh 1.0.0
```

Les rapports sont générés dans :

```text
build-reports/trivy
```

Exemples :

```text
api-gateway-trivy.json
api-gateway-trivy.sarif
member-service-trivy.json
member-service-trivy.sarif
front-trivy.json
front-trivy.sarif
```

Les rapports SARIF permettent une intégration dans des outils de sécurité et de revue de code.

Plan de remédiation général :

| Problème possible | Remédiation |
| --- | --- |
| Image de base vulnérable | Mettre à jour l’image de base |
| Dépendance vulnérable | Mettre à jour la dépendance applicative |
| Paquet système vulnérable | Rebuilder l’image avec une base corrigée |
| Vulnérabilité non exploitable localement | Documenter et surveiller |

---

## 12. Analyse Dive

Dive a été utilisé pour analyser les couches des images.

Script utilisé :

```text
scripts/analyze-dive.sh
```

Commande :

```bash
./scripts/analyze-dive.sh 1.0.0
```

Les rapports sont générés dans :

```text
build-reports/dive
```

Exemples :

```text
api-gateway-dive.json
member-service-dive.json
front-dive.json
dive-summary-1.0.0.md
```

L’analyse Dive permet de contrôler :

- le nombre de couches ;
- le contenu des couches ;
- l’efficacité de l’image ;
- la présence éventuelle de fichiers inutiles ;
- les pistes d’optimisation.

Optimisations retenues :

- utilisation d’images d’exécution légères ;
- séparation build / runtime ;
- exclusion des dossiers inutiles ;
- absence de `target`, `node_modules` et fichiers temporaires dans les images finales ;
- centralisation des exports dans `build-reports`.

---

# Partie B — Helm, Kubernetes, sécurité et GitOps

## 13. Objectifs de la Partie B

La Partie B porte sur :

- la création d’un chart Helm complet ;
- le déploiement Kubernetes de l’application ;
- la sécurisation du déploiement ;
- la gestion des secrets ;
- la configuration d’un mode production ;
- la mise en place d’une approche GitOps avec ArgoCD ;
- la démonstration d’une dérive et de sa correction automatique.

---

## 14. Chart Helm Odoru

Le chart Helm est disponible dans :

```text
infrastructure/helm/odoru
```

Structure principale :

```text
infrastructure/helm/odoru
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
└── templates
    ├── namespace.yaml
    ├── secrets.yaml
    ├── serviceaccount.yaml
    ├── networkpolicy.yaml
    ├── hpa.yaml
    ├── ingress.yaml
    ├── postgres.yaml
    ├── keycloak.yaml
    ├── config-server.yaml
    ├── discovery-service.yaml
    ├── api-gateway.yaml
    ├── microservices.yaml
    └── front.yaml
```

Le chart permet de déployer :

- le namespace `odoru` ;
- les secrets Kubernetes ;
- les bases PostgreSQL ;
- Keycloak ;
- Config Server ;
- Discovery Service ;
- API Gateway ;
- microservices métier ;
- frontend ;
- RBAC ;
- NetworkPolicy ;
- HPA ;
- Ingress.

---

## 15. Configuration standard et production

Deux fichiers de configuration sont disponibles.

Configuration standard :

```text
infrastructure/helm/odoru/values.yaml
```

Configuration production :

```text
infrastructure/helm/odoru/values-prod.yaml
```

La configuration production active notamment :

```text
NetworkPolicy
HPA
Ingress
RBAC
ServiceAccount
resources
probes
```

---

## 16. Sécurité Kubernetes

Les éléments de sécurité suivants sont présents dans le chart Helm.

| Élément | Statut | Description |
| --- | --- | --- |
| ServiceAccount | Implémenté | ServiceAccount dédié `odoru-app` |
| RBAC | Implémenté | Role et RoleBinding |
| NetworkPolicy | Implémenté | Politique default-deny et autorisations internes |
| Secrets Kubernetes | Implémenté | Secrets PostgreSQL et Keycloak |
| Resources | Implémenté | `requests` et `limits` CPU/mémoire |
| Probes | Implémenté | `readinessProbe` et `livenessProbe` |
| HPA | Implémenté | Autoscaling horizontal |
| Ingress | Implémenté | Ingress Traefik avec `odoru.local` |

---

## 17. Gestion des secrets

Les secrets Kubernetes sont générés par :

```text
infrastructure/helm/odoru/templates/secrets.yaml
```

Secrets créés :

```text
odoru-postgres-secret
odoru-keycloak-secret
```

Ils contiennent :

```text
POSTGRES_USER
POSTGRES_PASSWORD
KEYCLOAK_ADMIN
KEYCLOAK_ADMIN_PASSWORD
```

Ce rendu utilise des `Secret` Kubernetes, conformément au critère :

```text
Vault/ESO ou Secret Kubernetes
```

Un usage de Vault ou External Secrets Operator pourrait être envisagé comme amélioration future dans un environnement de production réel.

---

## 18. Probes et ressources

Tous les composants principaux disposent de :

- `readinessProbe` ;
- `livenessProbe` ;
- `resources.requests` ;
- `resources.limits`.

Les services Spring Boot utilisent :

```text
/actuator/health
```

Les bases PostgreSQL et Keycloak utilisent des probes TCP.

---

## 19. HPA

La configuration production génère des `HorizontalPodAutoscaler` pour :

```text
api-gateway
member-service
course-service
competition-service
badge-service
statistics-service
```

Paramètres principaux :

```yaml
minReplicas: 1
maxReplicas: 3
targetCPUUtilizationPercentage: 70
```

En environnement local Docker Desktop, les HPA peuvent afficher des avertissements si `metrics-server` n’est pas installé. Cela n’empêche pas la génération correcte des ressources Kubernetes.

---

## 20. Ingress

La configuration production génère un Ingress :

```text
odoru-ingress
```

Host configuré :

```text
odoru.local
```

Routes :

```text
/      -> front
/api   -> api-gateway
```

Le chart conserve également les NodePort pour faciliter les tests locaux :

```text
http://localhost:30081
http://localhost:30080
```

---

## 21. Validation Helm

Validation du chart standard :

```bash
helm lint infrastructure/helm/odoru
helm template odoru infrastructure/helm/odoru > /tmp/odoru-rendered.yaml
helm install odoru infrastructure/helm/odoru --dry-run --debug
```

Résultats obtenus :

```text
1 chart(s) linted, 0 chart(s) failed
STATUS: pending-install
```

Validation du chart avec `values-prod.yaml` :

```bash
helm lint infrastructure/helm/odoru -f infrastructure/helm/odoru/values-prod.yaml

helm template odoru infrastructure/helm/odoru \
  -f infrastructure/helm/odoru/values-prod.yaml \
  > /tmp/odoru-prod-rendered.yaml

helm install odoru infrastructure/helm/odoru \
  -f infrastructure/helm/odoru/values-prod.yaml \
  --dry-run --debug
```

Résultats obtenus :

```text
1 chart(s) linted, 0 chart(s) failed
STATUS: pending-install
```

La version production génère bien :

```text
NetworkPolicy
ServiceAccount
Role
RoleBinding
HorizontalPodAutoscaler
Ingress
```

---

## 22. Déploiement Kubernetes

Déploiement standard :

```bash
helm install odoru infrastructure/helm/odoru
```

Mise à jour :

```bash
helm upgrade odoru infrastructure/helm/odoru
```

Déploiement avec configuration production :

```bash
helm upgrade --install odoru infrastructure/helm/odoru \
  -f infrastructure/helm/odoru/values-prod.yaml
```

Vérification :

```bash
kubectl get pods -n odoru
kubectl get svc -n odoru
kubectl get deployments -n odoru
```

Résultat attendu :

```text
api-gateway          1/1 Running
front                1/1 Running
member-service       1/1 Running
course-service       1/1 Running
competition-service  1/1 Running
badge-service        1/1 Running
statistics-service   1/1 Running
config-server        1/1 Running
discovery-service    1/1 Running
keycloak             1/1 Running
postgres             1/1 Running
```

---

## 23. Automatisation Keycloak

La configuration Keycloak est automatisée par :

```text
scripts/setup-keycloak-odoru.sh
```

Ce script crée ou met à jour :

- le realm `odoru` ;
- le client `odoru-front` ;
- les rôles `MEMBER`, `SECRETARY`, `TEACHER`, `PRESIDENT` ;
- les utilisateurs de démonstration ;
- les mots de passe ;
- les redirections frontend.

Comptes créés :

| Utilisateur | Mot de passe | Rôles |
| --- | --- | --- |
| `lea.martin` | `secret123` | MEMBER |
| `sara.bernard` | `secret123` | MEMBER + SECRETARY |
| `marc.durand` | `secret123` | MEMBER + TEACHER |
| `paul.moreau` | `secret123` | MEMBER + PRESIDENT |

Commande :

```bash
kubectl port-forward -n odoru svc/keycloak 8090:8080
```

Puis, dans un autre terminal :

```bash
./scripts/setup-keycloak-odoru.sh
```

---

## 24. Seed des données métier

Les données métier sont initialisées par :

```text
scripts/seed-kubernetes-data.sh
```

Ce script vérifie ou crée les membres côté `member-service` afin que les comptes Keycloak correspondent aux utilisateurs présents en base métier.

Commande :

```bash
./scripts/seed-kubernetes-data.sh
```

Cette étape garantit la cohérence entre :

- les utilisateurs Keycloak ;
- les membres métier dans PostgreSQL.

---

## 25. Validation fonctionnelle

Après déploiement, l’application est accessible via :

```text
http://localhost:30081
```

Compte de test :

```text
lea.martin / secret123
```

La connexion passe par Keycloak, puis le frontend récupère les informations métier via l’API Gateway et les microservices.

Test API avec token Keycloak :

```bash
TOKEN=$(curl -s -X POST "http://localhost:8090/realms/odoru/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=odoru-front" \
  -d "username=lea.martin" \
  -d "password=secret123" | python3 -c "import sys,json; print(json.load(sys.stdin).get('access_token',''))")

curl -i -H "Authorization: Bearer $TOKEN" \
  http://localhost:30080/api/members/username/lea.martin
```

Résultat obtenu :

```text
HTTP/1.1 200 OK
```

---

# GitOps avec ArgoCD

## 26. Manifeste ArgoCD

Le manifeste ArgoCD est disponible dans :

```text
infrastructure/argocd/odoru-application.yaml
```

Contenu principal :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: odoru
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/hachimkidjei/odoru.git
    targetRevision: main
    path: infrastructure/helm/odoru
    helm:
      valueFiles:
        - values.yaml
        - values-prod.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: odoru

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 27. Installation ArgoCD

ArgoCD a été installé dans le namespace `argocd` :

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Vérification :

```bash
kubectl get pods -n argocd
```

Résultat observé :

```text
argocd-application-controller    1/1 Running
argocd-applicationset-controller 1/1 Running
argocd-dex-server                1/1 Running
argocd-notifications-controller  1/1 Running
argocd-redis                     1/1 Running
argocd-repo-server               1/1 Running
argocd-server                    1/1 Running
```

Vérification de la CRD `Application` :

```bash
kubectl api-resources | grep applications
```

Résultat :

```text
applications app,apps argoproj.io/v1alpha1 true Application
```

---

## 28. Application ArgoCD Odoru

Application du manifeste :

```bash
kubectl apply -f infrastructure/argocd/odoru-application.yaml
```

Vérification :

```bash
kubectl get application odoru -n argocd
```

Résultat obtenu :

```text
NAME    SYNC STATUS   HEALTH STATUS
odoru   Synced        Progressing
```

Le statut `Synced` confirme que l’état déclaré dans GitHub est synchronisé avec le cluster Kubernetes.

Le statut `Progressing` peut apparaître temporairement en environnement local, notamment à cause :

- du démarrage progressif des pods ;
- des probes récemment ajoutées ;
- de l’absence éventuelle de `metrics-server` pour les HPA.

---

## 29. Démonstration de dérive ArgoCD

Une dérive volontaire a été créée en modifiant manuellement le nombre de replicas du microservice `member-service` :

```bash
kubectl scale deployment member-service -n odoru --replicas=2
```

Grâce à la configuration :

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

ArgoCD a détecté la dérive et a réconcilié l’état réel du cluster avec l’état déclaré dans Git.

Preuve observée dans les événements ArgoCD :

```text
Updated sync status: Synced -> OutOfSync
Partial sync operation ... succeeded
Updated sync status: OutOfSync -> Synced
```

Vérification finale :

```bash
kubectl get application odoru -n argocd -o jsonpath='{.status.sync.status}{"\n"}'
kubectl get deployment member-service -n odoru -o jsonpath='{.spec.replicas}{"\n"}'
```

Résultat obtenu :

```text
Synced
1
```

Cela valide le mécanisme GitOps attendu : une modification manuelle du cluster est détectée comme dérive, puis corrigée automatiquement par ArgoCD grâce à `selfHeal`.

---

# Procédure complète de reproduction

## 30. Cloner le dépôt

```bash
git clone https://github.com/hachimkidjei/odoru.git
cd odoru
```

---

## 31. Construire les services Java

```bash
cd services/config-server && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/discovery-service && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/api-gateway && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/member-service && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/course-service && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/competition-service && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/badge-service && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/statistics-service && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
```

---

## 32. Construire le frontend

```bash
cd odoru-front
npm install
npm run build
cd ..
```

---

## 33. Construire les images OCI avec Buildah

```bash
chmod +x scripts/*.sh
./scripts/build-all.sh 1.0.0
```

Vérification :

```bash
buildah images | grep odoru
```

---

## 34. Analyser les images avec Trivy

```bash
./scripts/scan-trivy.sh 1.0.0
```

Rapports :

```text
build-reports/trivy
```

---

## 35. Analyser les images avec Dive

```bash
./scripts/analyze-dive.sh 1.0.0
```

Rapports :

```text
build-reports/dive
```

---

## 36. Charger les images dans Docker Desktop

```bash
docker load -i build-reports/oci/config-server-1.0.0.tar
docker load -i build-reports/oci/discovery-service-1.0.0.tar
docker load -i build-reports/oci/api-gateway-1.0.0.tar
docker load -i build-reports/oci/member-service-1.0.0.tar
docker load -i build-reports/oci/course-service-1.0.0.tar
docker load -i build-reports/oci/competition-service-1.0.0.tar
docker load -i build-reports/oci/badge-service-1.0.0.tar
docker load -i build-reports/oci/statistics-service-1.0.0.tar
docker load -i build-reports/oci/front-1.0.0.tar
```

---

## 37. Déployer avec Helm

```bash
helm upgrade --install odoru infrastructure/helm/odoru
```

Ou en configuration production :

```bash
helm upgrade --install odoru infrastructure/helm/odoru \
  -f infrastructure/helm/odoru/values-prod.yaml
```

Vérification :

```bash
kubectl get pods -n odoru
kubectl get svc -n odoru
```

---

## 38. Initialiser Keycloak et les données métier

Terminal 1 :

```bash
kubectl port-forward -n odoru svc/keycloak 8090:8080
```

Terminal 2 :

```bash
./scripts/setup-keycloak-odoru.sh
./scripts/seed-kubernetes-data.sh
```

---

## 39. Accéder à l’application

Frontend :

```text
http://localhost:30081
```

Compte de test :

```text
lea.martin / secret123
```

---

# Couverture des critères d’évaluation

## 40. Critères d’évaluation — Partie A

| Critère | Pondération |
| --- | --- |
| Analyse comparative Docker / Buildah | 15% |
| Chaîne de build Buildah fonctionnelle | 25% |
| Rapport Trivy + plan de remédiation | 20% |
| Analyse Dive + optimisations | 15% |
| Script de build intégré et documenté | 25% |

## 41. Couverture Partie A dans Odoru

| Critère | Couverture |
| --- | --- |
| Analyse comparative Docker / Buildah | Présente dans le README |
| Chaîne de build Buildah fonctionnelle | `scripts/build-all.sh` |
| Rapport Trivy + plan de remédiation | `build-reports/trivy` |
| Analyse Dive + optimisations | `build-reports/dive` |
| Script de build intégré et documenté | `build-all.sh`, `scan-trivy.sh`, `analyze-dive.sh` |

---

## 42. Critères d’évaluation — Partie B

| Critère | Pondération |
| --- | --- |
| Chart Helm complet et conforme | 30% |
| Déploiement Kubernetes avec sécurité : RBAC, NetworkPolicy, HPA | 25% |
| Gestion des secrets : Vault/ESO ou Secret Kubernetes | 20% |
| GitOps ArgoCD fonctionnel avec démonstration de dérive | 25% |

## 43. Couverture Partie B dans Odoru

| Critère | Couverture |
| --- | --- |
| Chart Helm complet et conforme | Chart `infrastructure/helm/odoru`, validé par `helm lint`, `helm template`, `dry-run` |
| Sécurité Kubernetes | ServiceAccount, RBAC, NetworkPolicy, HPA, resources, probes |
| Gestion des secrets | Secrets Kubernetes PostgreSQL et Keycloak |
| GitOps ArgoCD | Manifeste ArgoCD, sync automatique, `prune`, `selfHeal`, démo dérive `OutOfSync -> Synced` |

---

# Limites et choix techniques

## 44. Points d’attention liés à l’environnement local

Le déploiement a été réalisé dans un environnement Kubernetes local basé sur Docker Desktop. Certains comportements peuvent donc dépendre des composants disponibles dans ce cluster local.

Points d’attention identifiés :

- les `HorizontalPodAutoscaler` nécessitent un `metrics-server` pour exploiter les métriques CPU ;
- le statut `Health` ArgoCD peut apparaître temporairement `Progressing` pendant le démarrage ou le redéploiement des pods ;
- l’Ingress `odoru.local` nécessite un contrôleur Ingress compatible, par exemple Traefik, ainsi qu’une résolution locale du nom de domaine ;
- les secrets sont gérés avec des `Secret` Kubernetes pour répondre au périmètre du TP ;
- dans un contexte de production réel, une solution comme Vault ou External Secrets Operator serait plus adaptée pour la gestion centralisée des secrets.

Ces points correspondent à des choix d’environnement et de périmètre. Les ressources attendues pour le TP sont présentes dans le chart Helm et ont été validées par les commandes de rendu Helm et par ArgoCD.
---