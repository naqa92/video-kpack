---
runme:
  version: v3
---

---

## ⚙️ Prérequis

- **kp CLI** et **kubectl**

```bash
brew tap buildpacks-community/kpack-cli
brew install kp
```

- Une registry (ex: Docker Hub) + un token d’accès

- Un cluster Kubernetes

```bash
kind create cluster
```

- Registry & variables

```bash
export KPACK_VERSION="0.17.0"
export NS="demo"
export DOCKERHUB_USER="naqa92"
export DOCKERHUB_PAT="dckr_pat_xxx"
export APP="my-demo"
export VERSION="main"
export REPO="${DOCKERHUB_USER}/${APP}"
export GIT_URL="https://github.com/naqa92/video-kpack"
export GIT_REV="main"
export SUBPATH="app"
export SA="kp-default"
echo "Variables OK"
```

- Installation de **kpack** sur le cluster

```bash
kubectl apply -f https://github.com/buildpacks-community/kpack/releases/download/v${KPACK_VERSION}/release-${KPACK_VERSION}.yaml
k klock pods -n kpack
```

---

### 1. Namespace, Secret & ServiceAccount

```bash
k create namespace $NS

k -n $NS create secret docker-registry regcreds \
  --docker-username="$DOCKERHUB_USER" \
  --docker-password="$DOCKERHUB_PAT" \
  --docker-server="https://index.docker.io/v1/"

cat sa.yaml # ServiceAccount qui utilise le secret pour push/pull
k -n $NS apply -f sa.yaml

kp config default-service-account ${SA} --service-account-namespace $NS # Dire à kp: “utilise ce SA par défaut dans ce namespace”
kp config default-service-account # Voir la config active

kp config default-repository index.docker.io/${DOCKERHUB_USER}/kpack-repo # Dire à kp dans quel repo pousser ses artefacts (stores/builders/images)
kp config default-repository # Voir la config active
```

---

### 2. Store, Stack, Builder (Paketo + Python)

On va créer 3 ressources :

- ClusterStore : Ensemble de buildpacks disponible.
- ClusterStack : Images de base build/run.
- ClusterBuilder : L’assemblage (stack + store + ordre des buildpacks) que kpack utilisera.

**1. ClusterStore :**

```bash
kp clusterstore save default --buildpackage paketobuildpacks/python # push le store (qui contient les buildpacks) dans la registry.
```

> _Note: On peut ajouter d'autres buildpackages dans le même clusterstore (java, nodejs...)_

**2. ClusterStack :**

```bash
kp clusterstack save base \
  --build-image paketobuildpacks/build-jammy-base \
  --run-image paketobuildpacks/run-jammy-base
```

**3. ClusterBuilder :**

```bash
cat order-python.yaml
kp clusterbuilder save ${APP} \
  --tag index.docker.io/${REPO}-builder:latest \
  --stack base \
  --store default \
  --order ./order-python.yaml

kp clusterbuilder list
kp clusterbuilder status ${APP}
```

Permet de préparer le builder Paketo : base Jammy + buildpack Python. Pas de Dockerfile, le builder va détecter et builder tout seul.

---

### 3. Build d’une app Python d’exemple

Pourquoi : C’est l’objet piloté par kpack. Il déclenche le build, gère le cache, pousse l’image, et garde l’info du dernier digest.

```bash {"terminalRows":"25"}
kp image create ${APP} \
  --namespace $NS \
  --tag index.docker.io/${REPO}:${VERSION} \
  --git ${GIT_URL} \
  --git-revision ${GIT_REV} \
  --sub-path ${SUBPATH} \
  --cluster-builder ${APP} \
  --service-account ${SA}
```

```bash {"terminalRows":"15"}
kp build list --namespace $NS # suivre le build
kp build logs ${APP} --namespace $NS # voir les logs
```

Ce qui se passe : kpack crée un CRD Image, lance un Build, télécharge le code, détecte le langage, construit, met en cache, puis pousse index.docker.io/${REPO}:${VERSION}.

### 4. Récupérer le digest “latestImage”

Pourquoi : Le digest est la référence immuable à mettre dans les manifests YAML

```bash
kp image status ${APP} --namespace $NS | grep latestImage
```

Dans un workflow GitOps, c’est ce digest que l’Image Updater committe dans Git avant déploiement par Argo CD.

---

### 5. Rebuild automatique

kpack rebuild automatiquement sur trois types d’événements : Stack / Buildpacks / Source (Git).

Ici, on va voir le rebuild auto sur commit (Chaque commit → nouveau Build. Grâce au cache, c’est plus rapide que la 1ère fois)

Dans app/app.py, faire une modification et pousser :

```bash {"terminalRows":"16"}
ga ./app/app.py
gc -m "update"
gp
```

```bash {"terminalRows":"26"}
# Observer les builds
kp build list --namespace $NS
kp build logs ${APP} --namespace $NS
```

Pourquoi Image … not found à ANALYZE ? :

- 1er build : Normal, aucune image précédente → pas de cache.
- Builds suivants : Si l’image existe, kpack réutilise les couches (Python, pip, deps) ⇒ build plus rapide.

Si on met à jour le ClusterStack (nouvelles images base) ou le ClusterBuilder (nouveaux buildpacks), kpack reconstruit automatiquement les Image concernées.

kpack pousse à chaque build la même référence tag (ex. my-demo:main) mais avec un nouveau digest : C’est précisément là qu’Argo Image Updater (ou Flux Image Reflector) est utile : il modifie tes manifests Git pour pointer vers la nouvelle image, ce qui déclenche un rollout.

---

### Nettoyage

```bash {"terminalRows":"11"}
kp image delete ${APP} --namespace $NS
kp clusterbuilder delete ${APP}
kp clusterstack delete base
kp clusterstore delete default
kubectl -n $NS delete sa ${SA}
kubectl -n $NS delete secret regcreds
kubectl delete ns $NS
```
