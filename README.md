# Job Executor Continuous Delivery PoC

**Goal**: Create a simple but powerful Proof of Concept using Keptn Control-Plane and Job-Executor-Service to

* Deploy a service (using `helm upgrade`)
* Run performance tests (using `locust`) against this service


**Please note**: This is not a comprehensive tutorial, it is **not** covering every case regarding Keptn installation, Kubernetes, prometheus, nor job-executor-service.


## Pre Requisite

**You need access to a Kubernetes cluster with at least 4 vCPUs and 8 GB memory**

I recommend using Google Kubernetes Engine, but local setups with K3s/K3d should also work fine.


**Install Keptn 0.10.0 control-plane only**

```bash
curl -sL https://get.keptn.sh | KEPTN_VERSION=0.10.0 bash
keptn install --endpoint-service-type=LoadBalancer
```

**Install job-executor-service**

Experimental: Install from this PR: https://github.com/keptn-contrib/job-executor-service/pull/146

```bash
helm upgrade --install -n keptn job-executor-service https://raw.githubusercontent.com/christian-kreuzberger-dtx/keptn-job-executor-delivery-poc/main/job-executor/job-executor-service-0.1.5-dev-PR-146.tgz --set jobConfig.enableKubernetesApiAccess=true --wait
```

**Apply cluster role binding for job-executor-service**

This gives the job-executor-service role `job-executor-service` full `cluster-admin` access to your Kubernetes cluster. This is not recommended for production setups, but it is needed for this PoC to work (e.g., `helm upgrade` needs to be able to create namespaces, secrets, ...)

See [job-executor/clusterRoleBinding.yaml](job-executor/clusterRoleBinding.yaml) for details.

```bash
kubectl apply -f https://raw.githubusercontent.com/christian-kreuzberger-dtx/keptn-job-executor-delivery-poc/main/job-executor/clusterRoleBinding.yaml
```

**Install Prometheus Monitoring**

```bash
kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus --namespace monitoring --wait
```


**Install Keptns prometheus-service**
```bash
helm install -n keptn prometheus-service https://github.com/keptn-contrib/prometheus-service/releases/download/0.7.2/prometheus-service-0.7.2.tgz --wait
kubectl apply -f https://raw.githubusercontent.com/keptn-contrib/prometheus-service/release-0.7.2/deploy/role.yaml -n monitoring
```

## Project Setup

**Download or clone this repo**

```bash
git clone https://github.com/christian-kreuzberger-dtx/keptn-job-executor-delivery-poc.git
cd keptn-job-executor-delivery-poc
```

**Create Project and Service**

See [shipyard.yaml](shipyard.yaml) for details

```bash
PROJECT=podtato-head
keptn create project $PROJECT --shipyard=./shipyard.yaml
keptn create service helloservice --project=$PROJECT
```

**Add helm charts**

```bash
keptn add-resource --project=$PROJECT --service=helloservice --all-stages --resource=./helm/helloservice.tgz --resourceUri=charts/helloservice.tgz
```

**Add locust test files**

```bash
keptn add-resource --project=$PROJECT --service=helloservice --stage=qa --resource=./locust/basic.py
keptn add-resource --project=$PROJECT --service=helloservice --stage=qa --resource=./locust/locust.conf
```

**Add job-executor config**

```bash
keptn add-resource --project=$PROJECT --service=helloservice --all-stages --resource=job-executor-config.yaml --resourceUri=job/config.yaml
```

**Add SLO/SLI config files**

```bash
keptn add-resource --project=$PROJECT --service=helloservice --stage=qa --resource=prometheus/sli.yaml --resourceUri=prometheus/sli.yaml
keptn add-resource --project=$PROJECT --service=helloservice --stage=qa --resource=slo.yaml --resourceUri=slo.yaml
```

**Configure Prometheus Monitoring**

```bash
keptn configure monitoring prometheus --project=$PROJECT --service=helloservice
```

**Trigger delivery**

```bash
IMAGE="ghcr.io/podtato-head/podtatoserver"
VERSION=v0.1.1
SLOW_VERSION=v0.1.2

keptn trigger delivery --project=$PROJECT --service=helloservice --image=$IMAGE --tag=$VERSION --labels=version=$VERSION
```


**Trigger delivery of slow version**

```bash
keptn trigger delivery --project=$PROJECT --service=helloservice --image=$IMAGE --tag=$SLOW_VERSION --labels=version=$SLOW_VERSION,slow=true
```

## Cleanup

Once you're done, I recommend cleaning up:

**Delete project and resulting Kubernetes namespaces**

```bash
keptn delete project $PROJECT
kubectl delete namespace $PROJECT-qa $PROJECT-production
```

**Uninstall prometheus-service**
```bash
helm uninstall -n keptn prometheus-service
```

**Uninstall prometheus**
```bash
helm uninstall prometheus --namespace monitoring
```

**Remove role binding**
```bash
kubectl delete -f https://raw.githubusercontent.com/christian-kreuzberger-dtx/keptn-job-executor-delivery-poc/main/job-executor/clusterRoleBinding.yaml
```

**Uninstall job-executor-service**

```bash
helm uninstall -n keptn job-executor-service
```
