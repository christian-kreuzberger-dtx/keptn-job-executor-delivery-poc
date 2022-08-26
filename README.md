# Job Executor Continuous Delivery PoC

**Goal**: Create a simple but powerful Proof of Concept using Keptn Control-Plane and Job-Executor-Service to

* Deploy a service (using `helm upgrade`)
* Run performance tests (using `locust`) against this service


**Please note**: This is not a comprehensive tutorial, it is **not** covering every case regarding Keptn installation, Kubernetes, prometheus, nor job-executor-service.


## Pre Requisite

**You need access to a Kubernetes cluster with at least 4 vCPUs and 8 GB memory**

I recommend using Google Kubernetes Engine, but local setups with K3s/K3d should also work fine.


**Install Keptn 0.17.x control-plane only**

```bash
curl -sL https://get.keptn.sh | KEPTN_VERSION=0.17.0 bash
keptn install --endpoint-service-type=LoadBalancer
```

**Install job-executor-service**

Minimum version: 0.2.4

```bash

TASK_SUBSCRIPTION='sh.keptn.event.deployment.triggered\,sh.keptn.event.test.triggered\,sh.keptn.event.rollback.triggered'

helm upgrade --install --create-namespace -n keptn-jes \
  job-executor-service https://github.com/keptn-contrib/job-executor-service/releases/download/0.2.5-next.0/job-executor-service-0.2.5-next.0.tgz \
 --set remoteControlPlane.autoDetect.namespace="keptn",remoteControlPlane.autoDetect.enabled="true",remoteControlPlane.api.token="",remoteControlPlane.api.hostname="",remoteControlPlane.api.protocol="http"
```

**Kubernetes Role Based Access Control (RBAC) for helm deploy task**

By default the job-executor-service does not grant access to the Kubernetes api, the respective jobs (e.g., `helm install` or `helm upgrade`) would fail with `The connection to the server localhost:8080 was refused - did you specify the right host or port?`.

Therefore we have to define a service account for helm which allow the respective task/job to

* create namespaces, and
* run `helm install`, `helm upgrade` and `helm rollback` (which in return could create, modify and delete services, deployments, pods, secrets, configmaps, etc...).

One option to do this is to assign the job full `cluster-admin` access to your Kubernetes cluster, which we provide in
[job-executor/workloadClusterRoles.yaml](job-executor/workloadClusterRoles.yaml).

For more locked down setups, we recommend creating a dedicated Service Account and only link the needed roles/permissions depending on your Helm Chart. For more details, we recommend reading about [RBAC in the Kubernetes docs](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).


For now, we continue this example by using the full `cluster-admin` access:
```bash
kubectl apply -f https://raw.githubusercontent.com/christian-kreuzberger-dtx/keptn-job-executor-delivery-poc/main/job-executor/workloadClusterRoles.yaml
```

**Install Prometheus Monitoring**

```bash
kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus --namespace monitoring --wait
```


**Install Keptns prometheus-service**
```bash
helm install -n keptn prometheus-service https://github.com/keptn-contrib/prometheus-service/releases/download/0.8.5/prometheus-service-0.8.5.tgz --wait
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

**Add Remediation Config**

```bash
keptn add-resource --project=$PROJECT --service=helloservice --stage=production --resource=remediation.yaml
```

**Trigger delivery**

```bash
IMAGE="ghcr.io/podtato-head/podtatoserver"
VERSION=v0.1.1
SLOW_VERSION=v0.1.2

keptn trigger delivery --project=$PROJECT --service=helloservice --image=$IMAGE:$VERSION --labels=version=$VERSION
```

This should result in Version v0.1.1 being deployed to `qa` stage, with an `approval` waiting on `prod` stage. You can verify the result by executing
```bash
kubectl -n $PROJECT-qa get deployments -owide
kubectl -n $PROJECT-production get deployments -owide
```

**Trigger delivery of slow version**

```bash
keptn trigger delivery --project=$PROJECT --service=helloservice --image=$IMAGE:$SLOW_VERSION --labels=version=$SLOW_VERSION,slow=true
```

This should result in a failed quality gate in the `qa` stage, with a `rollback` triggered.

**Simulate a remediation**

In case of high-load, a remediation action would be to scale up the respective deployment. This can be simulated by sending a fake remediation.triggered event (which would be sent by prometheus-service):

```bash
keptn send event -f remediation.triggered.json
```

You should be able to verify the result by seeing multiple replicas in production:
```bash
kubectl -n $PROJECT-production get pods
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
kubectl delete -f https://raw.githubusercontent.com/christian-kreuzberger-dtx/keptn-job-executor-delivery-poc/main/job-executor/workloadClusterRoles.yaml
```

**Uninstall job-executor-service**

```bash
helm uninstall -n keptn-jes job-executor-service
```
