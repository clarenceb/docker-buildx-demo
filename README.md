Docker Buildx
=============

Note: Docker buildx only work with Linux container image builds at this time.

Download the latest linux arm64 `docker-buildx` binary release from [GitHub - https://github.com/docker/buildx](https://github.com/docker/buildx/releases/latest).

Rename the file to `docker-buildx` and copy to this repo's local directory.

Build the Docker-buildx builder image:

```sh
# Build the builder image which should have Docker client and Docker-buildx installed
docker build -t dockerbuildx . -f Dockerfile.buildx
docker tag <your-private-reg-fqdn>/dockerbuildx dockerbuildx
docker push <your-private-reg-fqdn>/dockerbuildx
```

or using ACR build tasks:

```sh
az acr build -r <your-acr-name> -t dockerbuildx -f Dockerfile.buildx .
```

Test docker buildx in Kubernetes:

```sh
# Create our interactive pod to test out Docker Buildx
kubectl create ns buildx

# Create a least privileged role that allows the pod to have access to deployments API in group Apps for Kubernetes otherwise you'll get this error:
# --> error: error while calling deploymentClient.Create for "builderxxxx0": deployments.apps is forbidden: User "system:serviceaccount:default:sa-cluster-admin" cannot create resource "deployments" in API group "apps" in the namespace "buildx"

# Workaround for demo, using cluster-admin.  Create your own role binding and service account for your pods.
kubectl create clusterrolebinding buildx-cluster-admin  --clusterrole=cluster-admin --serviceaccount=buildx:default

kubectl apply -f pod-builder.yaml -n buildx
kubectl get pod -n buildx

# Exec into the demo container
kubectl exec -ti builder -n buildx -- bash

# Inside container - create the buildx instance (3 replicas)
docker buildx create    \
  --driver kubernetes     \
  --driver-opt replicas=3 \
  --use \
  --name builderxxxx

docker buildx ls
docker buildx inspect builderxxxx

ls ~/.docker/buildx/

mkdir build
cd build

# Create a test Dockerfile
cat <<EOF>Dockerfile
FROM alpine:latest

LABEL description "Docker Buildx demo"
EOF

# Log into the container registry (requires admin user to be enabled or use azure workload idenity with acrPush role assigned)
docker login <your-private-reg-fqdn> -u <username> -p <password>

docker buildx build -t <your-private-reg>/containerbuild:001 --push .
```

Note: if the docker push doesn't work with azure workload idenity, you can try:

```sh
az login --identity
az acr login -n <registry>
```

but you'll need the Azure CLI in the container as well.
