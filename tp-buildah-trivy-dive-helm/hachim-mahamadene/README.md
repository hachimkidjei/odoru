L’adresse Keycloak utilisée par le frontend est donc :

http://localhost:8090
1. Récupérer le projet
git clone https://github.com/hachimkidjei/odoru.git
cd odoru

Si le projet est déjà cloné :

git pull
git status

Résultat attendu :

On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
2. Vérifier les prérequis
kubectl config use-context docker-desktop
kubectl get nodes

docker --version
helm version
buildah --version
trivy --version
dive --version
java -version
node -v
npm -v

Résultat attendu pour Kubernetes :

docker-desktop   Ready
3. Vérifier la configuration frontend

Le frontend utilise Vite. Les variables d’environnement nécessaires au build sont définies dans :

odoru-front/.env.production

Contenu attendu :

VITE_KEYCLOAK_URL=http://localhost:8090
VITE_KEYCLOAK_REALM=odoru
VITE_KEYCLOAK_CLIENT_ID=odoru-front
VITE_API_BASE_URL=http://localhost:30080

Ces valeurs ne sont pas des secrets. Elles servent uniquement à indiquer au frontend où trouver Keycloak et l’API Gateway en local.

Vérification :

cat odoru-front/.env.production
4. Construire le frontend
cd odoru-front
npm install
npm run build
cd ..

Vérification du build :

grep -R "localhost:8090\|odoru-front\|localhost:30080" -n odoru-front/dist/assets | head

Le résultat doit contenir des références à :

localhost:8090
odoru-front
localhost:30080
5. Construire les services Java
cd services/config-server && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/discovery-service && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/api-gateway && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/member-service && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/course-service && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/competition-service && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/badge-service && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
cd services/statistics-service && chmod +x mvnw && ./mvnw clean package -DskipTests && cd ../..
6. Construire les images avec Buildah
chmod +x scripts/*.sh
./scripts/build-all.sh 1.0.0

Vérification :

buildah images | grep odoru

Résultat attendu :

localhost/odoru/config-server
localhost/odoru/discovery-service
localhost/odoru/api-gateway
localhost/odoru/member-service
localhost/odoru/course-service
localhost/odoru/competition-service
localhost/odoru/badge-service
localhost/odoru/statistics-service
localhost/odoru/front
7. Transférer les images Buildah vers Docker Desktop

Docker Desktop Kubernetes utilise les images disponibles dans le moteur Docker local. Les images construites avec Buildah doivent donc être poussées vers Docker :

buildah push localhost/odoru/config-server:1.0.0 docker-daemon:localhost/odoru/config-server:1.0.0
buildah push localhost/odoru/discovery-service:1.0.0 docker-daemon:localhost/odoru/discovery-service:1.0.0
buildah push localhost/odoru/api-gateway:1.0.0 docker-daemon:localhost/odoru/api-gateway:1.0.0
buildah push localhost/odoru/member-service:1.0.0 docker-daemon:localhost/odoru/member-service:1.0.0
buildah push localhost/odoru/course-service:1.0.0 docker-daemon:localhost/odoru/course-service:1.0.0
buildah push localhost/odoru/competition-service:1.0.0 docker-daemon:localhost/odoru/competition-service:1.0.0
buildah push localhost/odoru/badge-service:1.0.0 docker-daemon:localhost/odoru/badge-service:1.0.0
buildah push localhost/odoru/statistics-service:1.0.0 docker-daemon:localhost/odoru/statistics-service:1.0.0
buildah push localhost/odoru/front:1.0.0 docker-daemon:localhost/odoru/front:1.0.0

Vérification :

docker images | grep odoru
8. Vérifier le chart Helm
helm lint infrastructure/helm/odoru
helm lint infrastructure/helm/odoru -f infrastructure/helm/odoru/values-prod.yaml

Résultat attendu :

1 chart(s) linted, 0 chart(s) failed
9. Déployer l’application avec Helm

Pour un premier déploiement ou une mise à jour :

helm upgrade --install odoru infrastructure/helm/odoru \
  -f infrastructure/helm/odoru/values-prod.yaml

Pour repartir d’un environnement propre :

helm uninstall odoru --ignore-not-found
kubectl delete namespace odoru --ignore-not-found

Attendre que le namespace soit supprimé :

kubectl get namespace odoru

Si la commande retourne NotFound, c’est normal : le namespace a bien été supprimé.

Puis redéployer :

helm upgrade --install odoru infrastructure/helm/odoru \
  -f infrastructure/helm/odoru/values-prod.yaml
10. Attendre le démarrage des services
kubectl rollout status deployment/config-server -n odoru --timeout=300s
kubectl rollout status deployment/discovery-service -n odoru --timeout=300s
kubectl rollout status deployment/keycloak -n odoru --timeout=600s
kubectl rollout status deployment/member-service -n odoru --timeout=300s
kubectl rollout status deployment/course-service -n odoru --timeout=300s
kubectl rollout status deployment/competition-service -n odoru --timeout=300s
kubectl rollout status deployment/badge-service -n odoru --timeout=300s
kubectl rollout status deployment/statistics-service -n odoru --timeout=300s
kubectl rollout status deployment/api-gateway -n odoru --timeout=300s
kubectl rollout status deployment/front -n odoru --timeout=300s

Vérification globale :

kubectl get pods -n odoru
kubectl get deployments -n odoru
kubectl get svc -n odoru

Résultat attendu :

Tous les pods        1/1 Running
Tous les deployments 1/1
11. Ouvrir l’accès local à Keycloak

Keycloak est exposé en interne dans Kubernetes avec un service ClusterIP. Pour l’utiliser depuis le navigateur et depuis les scripts locaux, ouvrir un port-forward :

kubectl port-forward -n odoru svc/keycloak 8090:8080

Garder ce terminal ouvert.

Vérification dans un deuxième terminal :

curl -i http://localhost:8090
curl -i http://localhost:8090/realms/master

Résultats attendus :

HTTP/1.1 302 Found

et :

HTTP/1.1 200 OK
12. Initialiser Keycloak

Dans un deuxième terminal, depuis la racine du projet :

./scripts/setup-keycloak-odoru.sh

Ce script initialise :

le realm odoru ;
les rôles MEMBER, SECRETARY, TEACHER, PRESIDENT ;
le client public odoru-front ;
les utilisateurs de démonstration.

Résultat attendu :

Token admin récupéré.
Realm odoru créé.
Rôle MEMBER créé.
Rôle SECRETARY créé.
Rôle TEACHER créé.
Rôle PRESIDENT créé.
Client odoru-front créé.
Token utilisateur OK.

Si les éléments existent déjà, le script les met à jour.

Vérifier le realm :

curl -i http://localhost:8090/realms/odoru/.well-known/openid-configuration

Résultat attendu :

HTTP/1.1 200 OK
13. Initialiser les données métier
./scripts/seed-kubernetes-data.sh

Ce script initialise les membres métier dans member-service.

Résultat attendu :

Seed métier terminé.
Comptes métier disponibles :
  lea.martin
  sara.bernard
  marc.durand
  paul.moreau
14. Tester l’authentification Keycloak
TOKEN=$(curl -s -X POST "http://localhost:8090/realms/odoru/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=odoru-front" \
  -d "username=lea.martin" \
  -d "password=secret123" | python3 -c "import sys,json; print(json.load(sys.stdin).get('access_token',''))")

echo "TOKEN_SIZE=${#TOKEN}"
echo "$TOKEN" | awk -F. '{print "JWT_PARTS=" NF}'

Résultat attendu :

TOKEN_SIZE=...
JWT_PARTS=3
15. Tester l’API Gateway
curl -i -H "Authorization: Bearer $TOKEN" \
  http://localhost:30080/api/members/username/lea.martin

Résultat attendu :

HTTP/1.1 200 OK

Exemple de réponse :

{
  "id": 1,
  "lastName": "Martin",
  "firstName": "Lea",
  "email": "lea.martin@example.com",
  "username": "lea.martin",
  "city": "Toulouse",
  "country": "France",
  "expertiseLevel": 1,
  "roles": ["MEMBER"]
}
16. Accéder à l’application Web

Garder le port-forward Keycloak actif :

kubectl port-forward -n odoru svc/keycloak 8090:8080

Puis ouvrir dans le navigateur :

http://localhost:30081

Si le navigateur garde une ancienne version du frontend en cache, faire :

CTRL + F5

ou ouvrir une fenêtre de navigation privée.

17. Comptes de démonstration
Utilisateur	Mot de passe	Rôles
lea.martin	secret123	MEMBER
sara.bernard	secret123	MEMBER + SECRETARY
marc.durand	secret123	MEMBER + TEACHER
paul.moreau	secret123	MEMBER + PRESIDENT

Exemple attendu après connexion avec sara.bernard :

Utilisateur : Sara Bernard
Username : @sara.bernard
ID membre : 2
Rôles : SECRETARY + MEMBER
Points importants
Le frontend et l’API Gateway sont exposés par des services NodePort.
Keycloak reste interne au cluster et doit être rendu accessible localement par kubectl port-forward.
Le port-forward Keycloak doit rester ouvert pendant l’utilisation de l’application.
Les scripts setup-keycloak-odoru.sh et seed-kubernetes-data.sh doivent être exécutés après un déploiement propre, car le namespace et les bases sont recréés.
Les variables du fichier odoru-front/.env.production sont nécessaires au build du frontend. Sans elles, le frontend construit des URLs Keycloak invalides.

Tu peux ajouter juste avant cette section une phrase simple :

```markdown
La procédure ci-dessous a été testée sur deux postes différents avec Docker Desktop Kubernetes. Elle permet de reconstruire les images, de déployer les ressources Kubernetes avec Helm, d’initialiser Keycloak, d’injecter les données métier et d’accéder à l’application via le navigateur.

Et pour être cohérent avec ce que tu as constaté : ne parle pas de PowerShell comme obligation. Tu peux éventuellement ajouter une note courte :

Sous Windows avec WSL, le port-forward peut être lancé depuis WSL. Si le navigateur ne voit pas `localhost:8090`, il est aussi possible de lancer le même port-forward depuis PowerShell.

Mais dans la procédure principale, garde seulement WSL.
