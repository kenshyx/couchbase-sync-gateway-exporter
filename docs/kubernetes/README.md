# Deploying to Kubernetes

## Install Helm

```sh
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller --upgrade
```

## Install Couchbase Operator

> [Docs](https://docs.couchbase.com/operator/current/helm-setup-guide.html)

```sh
helm repo add couchbase https://couchbase-partners.github.io/helm-charts/

helm install --namespace couchbase --name couchbase-operator couchbase/couchbase-operator
helm install --namespace couchbase --name couchbase couchbase/couchbase-cluster -f kubernetes/couchbase/values.yaml
```

This should install the operator and launch a new Couchbase cluster.

It should create a Couchbase "cluster" with a single node, a `default`
bucket and credentials being `Administrator`/`password`.

You can port forward to it to see if everything is good (it may take a while):

```sh
kubectl port-forward -n couchbase svc/couchbase-couchbase-cluster-ui 8091:8091
open http://localhost:8091
```

## Install Prometheus Operator

> [Docs](https://github.com/helm/charts/tree/master/stable/prometheus-operator)

```sh
helm install --namespace prometheus --name prom stable/prometheus-operator -f kubernetes/prometheus/values.yaml
```

This should show you all pods running:

```sh
kubectl -n prometheus get pods
```

## Setup Sync Gateway + Exporter

Create the config secret:

```sh
kubectl create -n couchbase secret generic sgw-config --from-file=./kubernetes/sgw-config.json
kubectl apply -f kubernetes/sgw
```

This should launch 2 SGW instances each one with the exporter as a sidecar:

```sh
kubectl -n couchbase get pods -l app=sync-gateway
```

## Checking things

```sh
kubectl -n prometheus port-forward svc/prom-prometheus-operator-prometheus 9090:9090

open http://localhost:9090
```

You should see something like this:

![](/docs/kubernetes/screen-1sgw.png)

Then we can scale the sync gateway with:

```sh
kubectl scale -n couchbase deploy/sync-gateway --replicas 2
```

And refresh that page, so you can see something like this:


![](/docs/kubernetes/screen-2sgw.png)

## Grafana

First, lets port-forward grafana to our local environment:

```sh
kubectl -n prometheus port-forward svc/prom-grafana 3000:80

open http://localhost:3000
```

Username and password are `admin` and `admin`. It will have a
set of dashboards already there.

To install our dashboard, you can run:

```sh
make grafana
```

And then you should be able to find the Sync Gateway dashboard on
http://localhost:3000:

![](/docs/kubernetes/dash.png)

By default it will show metrics for all Sync Gateway instances, but you can
of course filter them:

![](/docs/kubernetes/choose-instances.png)

Those values are queried on dashboard load, so you may need to reload it as
new instances come online.

By default, it will use the pod name to differentiate between Sync Gateway
instances. You can revert back to the prometheus-operator default of using
the pod IP by commenting
[these lines](https://github.com/caarlos0/couchbase-sync-gateway-exporter/blob/master/kubernetes/sgw/servicemonitor.yaml#L18-L25)
and re-applying that file.