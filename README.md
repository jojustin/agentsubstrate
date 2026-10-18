# Running and Resuming a Stateful Workload with Agent Substrate

## Pre-requisite 

Install Podman -> brew install podman
Verify the installation -> podman --version
set the cpu -> podman machine set --cpus 4 --memory 8192
Initialize the Podman machine -> podman machine init
Start the machine -> podman machine start
Verify its status -> podman machine list

## Install Kind

brew install kind kubectl
kind version
kubectl version --client
podman machine stop
export KIND_EXPERIMENTAL_PROVIDER=podman
podman machine start

## Create cluster 
kind create cluster --name kind
kind get clusters
kubectl cluster-info --context kind-kind
kubectl get nodes
kubectl get pods -A


## Test the cluster deployment
kubectl create deployment nginx --image=nginx
kubectl get deployments
kubectl get pods
kubectl delete deployment nginx


## Clone Agent Substrate repository
Clone the repository -> git clone https://github.com/agent-substrate/substrate.git

cd substrate
Verify repository -> ls
Record the repository version -> git rev-parse --short HEAD

## Configure Docker CLI compatibility
mkdir -p "$HOME/.local/substrate-tools"

cat > "$HOME/.local/substrate-tools/docker" <<'EOF'
#!/usr/bin/env bash
exec podman "$@"
EOF


chmod +x "$HOME/.local/substrate-tools/docker"

export PATH="$HOME/.local/substrate-tools:$PATH"

docker --version

## Create the Substrate Kind cluster
export KIND_EXPERIMENTAL_PROVIDER=podman
export KIND_CLUSTER_NAME=kind
export ATE_ATENET_DATAPLANE=agentgateway
export ATE_USE_PODMAN=true
export KO_DOCKER_REPO=localhost:5001
export KO_DEFAULTPLATFORMS=linux/arm64

./hack/create-kind-cluster.sh

## Verify Substrate Kind cluster
kubectl config current-context
kubectl get nodes
kubectl get pods -A

## Create registry 
podman run -d --name kind-registry --network kind -p 127.0.0.1:5001:5000 docker.io/library/registry:3
Verify -> 
podman ps --filter name=kind-registry
curl -4 http://127.0.0.1:5001/v2/

## Install Agent Substrate runtime
./hack/kind.sh export kubeconfig --name="kind"
go run ./cmd/ate-setup deploy ate-system --config /Users/joyce/Library/Caches/ate-setup/failed/kind-kind-20261018T064845Z.yaml

<!-- ./bin/ate-setup deploy ate-system --config /Users/joyce/Library/Caches/ate-setup/failed/kind-kind-20261018T070846Z.yaml -->

./hack/install-ate-kind.sh --deploy-ate-system --credential-provider='{"name":"k8s.io"}'


## Verify the deployment
kubectl get pods -n ate-system

kubectl get deployments -n ate-system

kubectl get svc -n ate-system


## Install the Stateful Counter demo
verify cpuset is enabled -> podman info --format '{{json .Host.CgroupControllers}}'
response should contain - ["cpuset","cpu","io","memory","pids"]

start the existing Kind node if it is stopped -> podman start kind-control-plane
Then check -> podman exec kind-control-plane  cat /sys/fs/cgroup/cgroup.controllers

Start the registry if its not already -> podman start kind-registry
Verify -> podman ps --filter name=kind-registry

./hack/install-ate-kind.sh --deploy-demo-counter

### Install the Substrate CLI
go install ./cmd/kubectl-ate

export PATH="$(go env GOPATH)/bin:$PATH"

kubectl ate --help

### Create your counter actor
kubectl ate create actor my-counter-1 -a ate-demo-counter --template counter

## Access the counter 
kubectl port-forward -n ate-system svc/atenet-router 8000:80

In another terminal try to access -> curl -X POST -H "ate-target-actor: ate-demo-counter/my-counter-1" -i http://localhost:8000/
