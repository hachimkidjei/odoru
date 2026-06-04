Hachim Kidjei MAHAMADENE — Velat TAS

## 0. Contexte du rendu

Ce rendu correspond au TP **Buildah, Trivy, Dive & Helm/Kubernetes**.

Le sujet de référence mentionne l’application **MIAGE Bank**. Dans ce rendu, le même travail technique a été appliqué à mon projet microservices **Odoru**, qui possède une architecture comparable :

- frontend web ;
- API Gateway ;
- microservices métier ;
- bases PostgreSQL ;
- Config Server ;
- Discovery Service ;
- authentification avec Keycloak ;
- déploiement Kubernetes avec Helm.

Dépôt GitHub du projet Odoru :

```text
https://github.com/hachimkidjei/odoru
```

Objectif du rendu :

```text
Buildah → Trivy → Dive → Helm → Kubernetes → Tests applicatifs
```

---

# 1. Présentation du projet Odoru

Odoru est une application de gestion d’un club de danse.

Elle comprend les composants suivants :

```text
odoru-front
api-gateway
config-server
discovery-service
member-service
course-service
competition-service
badge-service
statistics-service
keycloak
postgresql
```

Architecture déployée :

```text
Navigateur
   |
   | http://localhost:30081
   v
Frontend React / Vite
   |
   | http://localhost:30080
   v
API Gateway
   |
   +--> member-service:8081
   +--> course-service:8082
   +--> competition-service:8083
   +--> badge-service:8084
   +--> statistics-service:8085

Services techniques :
   - config-server:8888
   - discovery-service:8761
   - keycloak:8080

Bases PostgreSQL :
   - member-postgres:5432
   - course-postgres:5432
   - competition-postgres:5432
   - badge-postgres:5432
```

---

# Partie A — Buildah, Trivy et Dive

## A.1. Analyse comparative Docker / Buildah

Docker est souvent utilisé pour construire et exécuter des images de conteneurs. Buildah permet de construire des images compatibles OCI sans dépendre directement d’un démon Docker.

Dans ce TP, Buildah est utilisé pour construire les images applicatives Odoru.

Comparaison synthétique :

| Critère | Docker | Buildah |
|---|---|---|
| Construction d’image | Oui | Oui |
| Démon nécessaire | Oui, Docker daemon | Non |
| Format d’image | Docker / OCI | OCI |
| Usage CI/CD | Possible | Très adapté |
| Séparation build / runtime | Moins nette | Plus nette |
| Approche rootless | Possible | Très adaptée |

Buildah est pertinent ici car il permet de produire des images OCI de manière plus légère, scriptable et adaptée à une chaîne DevSecOps.

---

## A.2. Services conteneurisés

Les composants Odoru possèdent chacun un `Containerfile`.

Fichiers utilisés :

```text
odoru-front/Containerfile
services/api-gateway/Containerfile
services/badge-service/Containerfile
services/competition-service/Containerfile
services/config-server/Containerfile
services/course-service/Containerfile
services/discovery-service/Containerfile
services/member-service/Containerfile
services/statistics-service/Containerfile
```

Exemple de construction d’image :

```bash
buildah bud -f services/member-service/Containerfile -t odoru/member-service:1.0.0 .
```

Images construites :

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

Vérification :

```bash
buildah images
```

---

## A.3. Export des images

Les images ont été exportées sous forme d’archives afin de pouvoir être :

- scannées avec Trivy ;
- analysées avec Dive ;
- chargées dans Docker Desktop pour Kubernetes.

Exemple :

```bash
buildah push \
  localhost/odoru/member-service:1.0.0 \
  docker-archive:build-reports/oci/member-service-1.0.0.tar:localhost/odoru/member-service:1.0.0
```

Les archives sont générées dans :

```text
build-reports/oci/
```

Ce dossier est volontairement exclu de Git car les archives sont lourdes.

---

## A.4. Analyse de vulnérabilités avec Trivy

Trivy a été utilisé pour analyser les vulnérabilités des images Odoru.

Script fourni :

```text
scripts/scan-trivy.sh
```

Commande d’exécution :

```bash
./scripts/scan-trivy.sh 1.0.0
```

Les rapports sont générés dans :

```text
build-reports/trivy/
```

Rapports produits :

```text
api-gateway-trivy.json
api-gateway-trivy.sarif
badge-service-trivy.json
badge-service-trivy.sarif
competition-service-trivy.json
competition-service-trivy.sarif
config-server-trivy.json
config-server-trivy.sarif
course-service-trivy.json
course-service-trivy.sarif
discovery-service-trivy.json
discovery-service-trivy.sarif
front-trivy.json
front-trivy.sarif
member-service-trivy.json
member-service-trivy.sarif
statistics-service-trivy.json
statistics-service-trivy.sarif
```

Exemple de commande Trivy :

```bash
trivy image \
  --input build-reports/oci/member-service-1.0.0.tar \
  --severity HIGH,CRITICAL \
  --format json \
  --output build-reports/trivy/member-service-trivy.json
```

---

## A.5. Plan de remédiation Trivy

Les rapports Trivy permettent d’identifier les vulnérabilités présentes dans les images.

Plan de remédiation retenu :

1. utiliser des images de base maintenues ;
2. reconstruire régulièrement les images ;
3. mettre à jour les dépendances Maven et npm ;
4. surveiller les vulnérabilités `HIGH` et `CRITICAL` ;
5. conserver les rapports JSON et SARIF pour la traçabilité ;
6. intégrer à terme le scan Trivy dans une chaîne CI/CD.

Dans ce rendu, l’objectif principal était de produire les rapports et de démontrer l’intégration de Trivy dans la chaîne de build.

---

## A.6. Analyse des images avec Dive

Dive a été utilisé pour analyser les couches des images.

Script fourni :

```text
scripts/analyze-dive.sh
```

Commande :

```bash
./scripts/analyze-dive.sh 1.0.0
```

Rapports générés :

```text
build-reports/dive/
```

Contenu obtenu :

```text
api-gateway-dive.json
badge-service-dive.json
competition-service-dive.json
config-server-dive.json
course-service-dive.json
discovery-service-dive.json
front-dive.json
member-service-dive.json
statistics-service-dive.json
dive-summary-1.0.0.md
```

Objectifs de l’analyse Dive :

- observer les couches des images ;
- identifier les fichiers ajoutés ;
- vérifier l’efficacité des images ;
- repérer des pistes d’optimisation.

---

## A.7. Optimisations possibles des images

Pistes d’optimisation possibles :

- utiliser des images de base plus minimales ;
- éviter de copier des fichiers inutiles dans l’image ;
- conserver uniquement le livrable final ;
- exclure les dossiers `target`, `node_modules`, `.git`, `.idea` ;
- utiliser `.containerignore` ;
- reconstruire régulièrement les images de base ;
- séparer clairement build et runtime.

---

# Partie B — Helm et Kubernetes

## B.1. Chart Helm

Un chart Helm complet a été créé pour Odoru.

Emplacement :

```text
infrastructure/helm/odoru/
```

Structure :

```text
infrastructure/helm/odoru
├── Chart.yaml
├── charts
├── templates
│   ├── _helpers.tpl
│   ├── api-gateway.yaml
│   ├── config-server.yaml
│   ├── discovery-service.yaml
│   ├── front.yaml
│   ├── keycloak.yaml
│   ├── microservices.yaml
│   ├── namespace.yaml
│   ├── postgres.yaml
│   └── secrets.yaml
└── values.yaml
```

Validation du rendu Helm :

```bash
helm template odoru infrastructure/helm/odoru
```

---

## B.2. Ressources Kubernetes créées

Le chart Helm déploie :

```text
Namespace
Secrets
Deployments
Services
PostgreSQL
Keycloak
Config Server
Discovery Service
Microservices métier
API Gateway
Frontend
```

Namespace utilisé :

```text
odoru
```

Vérification :

```bash
kubectl get ns
kubectl get pods -n odoru
kubectl get svc -n odoru
kubectl get all -n odoru
```

---

## B.3. Déploiement Helm

Installation :

```bash
helm install odoru infrastructure/helm/odoru
```

Mise à jour :

```bash
helm upgrade odoru infrastructure/helm/odoru
```

Vérification :

```bash
helm list
helm status odoru
```

Résultat obtenu :

```text
NAME    NAMESPACE   STATUS      CHART       APP VERSION
odoru   default     deployed    odoru-0.1.0 1.0.0
```

---

## B.4. Vérification du cluster Kubernetes

Contexte Kubernetes utilisé :

```bash
kubectl config current-context
```

Résultat :

```text
docker-desktop
```

Nœud Kubernetes :

```bash
kubectl get nodes
```

Résultat :

```text
NAME             STATUS   ROLES           VERSION
docker-desktop   Ready    control-plane   v1.32.2
```

---

## B.5. Pods déployés

Commande :

```bash
kubectl get pods -n odoru
```

Résultat obtenu :

```text
api-gateway                 1/1 Running
badge-postgres              1/1 Running
badge-service               1/1 Running
competition-postgres        1/1 Running
competition-service         1/1 Running
config-server               1/1 Running
course-postgres             1/1 Running
course-service              1/1 Running
discovery-service           1/1 Running
front                       1/1 Running
keycloak                    1/1 Running
member-postgres             1/1 Running
member-service              1/1 Running
statistics-service          1/1 Running
```

---

## B.6. Services Kubernetes

Commande :

```bash
kubectl get svc -n odoru
```

Services exposés :

```text
api-gateway          NodePort    8080:30080/TCP
front                NodePort    80:30081/TCP
member-service       ClusterIP   8081/TCP
course-service       ClusterIP   8082/TCP
competition-service  ClusterIP   8083/TCP
badge-service        ClusterIP   8084/TCP
statistics-service   ClusterIP   8085/TCP
config-server        ClusterIP   8888/TCP
discovery-service    ClusterIP   8761/TCP
```

Choix d’architecture :

- `front` est exposé en `NodePort` ;
- `api-gateway` est exposée en `NodePort` ;
- les microservices restent internes en `ClusterIP` ;
- les bases PostgreSQL restent internes en `ClusterIP`.

---

## B.7. API Gateway et routage Kubernetes

Une correction importante a été réalisée sur l’API Gateway.

Initialement, les routes utilisaient :

```yaml
uri: lb://member-service
```

En environnement Kubernetes, cette configuration a provoqué des erreurs DNS car la Gateway essayait de résoudre des noms de pods.

La configuration a été adaptée pour utiliser directement les Services Kubernetes :

```yaml
uri: http://member-service:8081
uri: http://course-service:8082
uri: http://competition-service:8083
uri: http://badge-service:8084
uri: http://statistics-service:8085
```

Ainsi, le load balancing est assuré par Kubernetes via les Services `ClusterIP`.

Vérification :

```bash
kubectl port-forward -n odoru svc/config-server 8888:8888
```

Puis :

```bash
curl http://localhost:8888/api-gateway/default | grep -E "uri|member-service|lb://"
```

Résultat attendu :

```text
spring.cloud.gateway.routes[0].uri = http://member-service:8081
```

Il ne doit plus y avoir de route `lb://`.

---

## B.8. Config Server

Le Config Server fonctionne en mode `native`.

Dans Kubernetes, il lit la configuration depuis :

```text
/config-repo
```

La configuration est embarquée dans l’image `config-server`.

Variable utilisée :

```yaml
SPRING_CLOUD_CONFIG_SERVER_NATIVE_SEARCH_LOCATIONS=file:/config-repo
```

Cela permet aux microservices de récupérer une configuration adaptée au cluster Kubernetes.

---

## B.9. Gestion des secrets

Le chart Helm crée des secrets Kubernetes pour PostgreSQL et Keycloak.

Exemples :

```text
odoru-postgres-secret
odoru-keycloak-secret
```

Ces secrets contiennent notamment :

```text
POSTGRES_USER
POSTGRES_PASSWORD
KEYCLOAK_ADMIN
KEYCLOAK_ADMIN_PASSWORD
```

Dans cette version, les secrets restent simples pour un environnement de TP local.

Amélioration possible :

```text
Vault
External Secrets Operator
```

---

## B.10. Authentification Keycloak

L’application utilise Keycloak pour l’authentification.

Realm :

```text
odoru
```

Client :

```text
odoru-front
```

Rôles :

```text
MEMBER
SECRETARY
TEACHER
PRESIDENT
```

La configuration Keycloak est automatisée par le script :

```text
scripts/setup-keycloak-odoru.sh
```

Ce script crée ou met à jour :

```text
realm odoru
client odoru-front
rôles MEMBER / SECRETARY / TEACHER / PRESIDENT
utilisateurs de démonstration
mots de passe
redirect URIs
web origins
```

Comptes de démonstration :

```text
lea.martin / secret123       -> MEMBER
sara.bernard / secret123     -> MEMBER + SECRETARY
marc.durand / secret123      -> MEMBER + TEACHER
paul.moreau / secret123      -> MEMBER + PRESIDENT
```

Commande :

```bash
./scripts/setup-keycloak-odoru.sh
```

---

## B.11. Données métier de test

Keycloak gère l’identité, les mots de passe et les rôles JWT.

Le service `member-service`, lui, gère les profils métier Odoru.

Le lien entre les deux repose sur le champ :

```text
username
```

Pour rendre le projet reproductible, un script crée les profils métier dans la base PostgreSQL Kubernetes :

```text
scripts/seed-kubernetes-data.sh
```

Commande :

```bash
./scripts/seed-kubernetes-data.sh
```

Ce script crée les profils :

```text
lea.martin
sara.bernard
marc.durand
paul.moreau
```

Tous les profils métier sont créés comme membres du club dans `member-service`.

Les responsabilités supplémentaires `SECRETARY`, `TEACHER` et `PRESIDENT` sont portées par les rôles Keycloak dans le token JWT.

---

# 2. Procédure complète de reproduction

## 2.1. Construire les services

Depuis la racine du projet :

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

Build frontend :

```bash
cd odoru-front
npm install
npm run build
cd ..
```

---

## 2.2. Construire les images avec Buildah

Exemple :

```bash
buildah bud -f services/member-service/Containerfile -t odoru/member-service:1.0.0 .
```

Même principe pour les autres services.

---

## 2.3. Scanner avec Trivy

```bash
./scripts/scan-trivy.sh 1.0.0
```

---

## 2.4. Analyser avec Dive

```bash
./scripts/analyze-dive.sh 1.0.0
```

---

## 2.5. Charger les images dans Docker Desktop

Exemple :

```bash
docker load -i build-reports/oci/member-service-1.0.0.tar
```

Même principe pour les autres images.

---

## 2.6. Déployer avec Helm

```bash
helm install odoru infrastructure/helm/odoru
```

Ou si la release existe déjà :

```bash
helm upgrade odoru infrastructure/helm/odoru
```

---

## 2.7. Configurer Keycloak automatiquement

```bash
./scripts/setup-keycloak-odoru.sh
```

---

## 2.8. Créer les données métier

```bash
./scripts/seed-kubernetes-data.sh
```

---

# 3. Tests de validation

## 3.1. Test sans token

```bash
curl -i http://localhost:30080/api/members
```

Résultat attendu :

```text
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer
```

Ce résultat est normal : l’API Gateway protège les routes.

---

## 3.2. Récupérer un token Keycloak

```bash
TOKEN=$(curl -s -X POST "http://localhost:8090/realms/odoru/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=odoru-front" \
  -d "username=lea.martin" \
  -d "password=secret123" | python3 -c "import sys,json; print(json.load(sys.stdin).get('access_token',''))")
```

Vérification :

```bash
echo ${#TOKEN}
```

Résultat obtenu :

```text
1441
```

---

## 3.3. Test avec token

```bash
curl -i -H "Authorization: Bearer $TOKEN" \
  http://localhost:30080/api/members/username/lea.martin
```

Résultat obtenu :

```text
HTTP/1.1 200 OK
```

Réponse obtenue :

```json
{
  "id": 1,
  "lastName": "Martin",
  "firstName": "Lea",
  "email": "lea.martin@example.com",
  "username": "lea.martin",
  "city": "Toulouse",
  "country": "France",
  "expertiseLevel": 1,
  "registrationStatus": "PENDING_REVIEW",
  "membershipFeePaid": false,
  "medicalCertificateProvided": false,
  "registrationCheckedBySecretary": false,
  "roles": ["MEMBER"]
}
```

Ce test valide la chaîne complète :

```text
Client
-> API Gateway exposée en NodePort
-> validation JWT Keycloak
-> member-service
-> PostgreSQL Kubernetes
```

---

## 3.4. Accès frontend

Adresse :

```text
http://localhost:30081
```

Compte de test :

```text
lea.martin / secret123
```

Autres comptes :

```text
sara.bernard / secret123
marc.durand / secret123
paul.moreau / secret123
```

---

# 4. Historique Git

Commits principaux :

```text
27793d1 Add automated Keycloak and seed setup for Odoru
9ff53f0 Add Kubernetes Helm deployment for Odoru
c0c239e Add Dive image analysis reports
2a340bd Add complete Trivy scans for Odoru images
9cee0f3 Initial Odoru project with Buildah containerization and Trivy reports
```

État final :

```text
On branch main
Your branch is up to date with 'origin/main'.
nothing to commit, working tree clean
```

---

# 5. Limites et perspectives

Fonctionnalités réalisées :

```text
Buildah
Trivy
Dive
Helm
Kubernetes
Keycloak
API Gateway
PostgreSQL
Scripts d’automatisation Keycloak et seed métier
```

Éléments non implémentés dans cette version mais identifiés comme perspectives :

```text
ArgoCD
Vault
External Secrets Operator
NetworkPolicy
RBAC avancé
Traefik
Ingress
```

Améliorations possibles :

- ajouter ArgoCD pour un déploiement GitOps ;
- remplacer les secrets simples par Vault ou External Secrets ;
- ajouter des NetworkPolicies ;
- ajouter des règles RBAC plus strictes ;
- exposer l’application via Traefik / Ingress ;
- ajouter des volumes persistants pour PostgreSQL ;
- ajouter des probes Kubernetes `livenessProbe` et `readinessProbe` ;
- automatiser encore davantage la génération des images.

---

# 6. Bibliographie

- Buildah — Documentation officielle : https://buildah.io/
- Buildah — Dépôt GitHub : https://github.com/containers/buildah
- OCI Image Format Specification : https://github.com/opencontainers/image-spec
- Trivy — Documentation officielle : https://aquasecurity.github.io/trivy/
- Dive — Dépôt GitHub : https://github.com/wagoodman/dive
- Helm — Documentation officielle : https://helm.sh/docs/
- Kubernetes — Documentation officielle : https://kubernetes.io/docs/
- ArgoCD — Documentation officielle : https://argo-cd.readthedocs.io/
- HashiCorp Vault — Documentation : https://developer.hashicorp.com/vault
- External Secrets Operator : https://external-secrets.io/
- Kubernetes — NetworkPolicy : https://kubernetes.io/docs/concepts/services-networking/network-policies/
- Kubernetes — RBAC : https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- Traefik — Documentation officielle : https://doc.traefik.io/traefik/

---

# 7. Conclusion

Ce rendu démontre l’adaptation complète du TP Buildah, Trivy, Dive et Helm/Kubernetes au projet microservices Odoru.

Les images ont été construites avec Buildah, analysées avec Trivy et Dive, puis déployées dans Kubernetes avec Helm.

L’application fonctionne avec :

- un frontend exposé en NodePort ;
- une API Gateway sécurisée par Keycloak ;
- des microservices internes en ClusterIP ;
- des bases PostgreSQL dans Kubernetes ;
- une configuration centralisée via Config Server ;
- des scripts permettant de reproduire la configuration Keycloak et les données métier.

Le test final avec token Keycloak retourne `200 OK`, ce qui valide la chaîne complète de bout en bout.
