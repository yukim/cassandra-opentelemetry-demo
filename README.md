# cassandra-opentelemetry-demo

The repository for Apache Cassandra OpenTelemetry integration demo.

* `cassandra/`
    * Dockerfile to build Apache Cassandra container image with OpenTelemetry integration from https://github.com/yukim/cassandra `cassandra-4.0-otel` branch.
* `k8s/`
    * Resource definitions for the demo
* `management-api-for-cassandra/`
    * Dockerfile to build container image from https://github.com/k9ssange/management-api-for-cassandra to include Apache Cassandra with OpenTelemetry integration.

# Tested demo environment

* AWS EKS deployed on ap-northeast-1 region
    * 6 x t3.xlarge (4 vCPU / 16GiB mem) across 3 AZ
        * 3 nodes dedicated for Cassandra
        * other 3 nodes for apps / opentelemetry collector

The demo environment can be created in ap-northeast-1 region with eksctl:

```
eksctl create cluster -f eks/cluster.yaml
```

# Run demo on Kubernetes

## 1. Install cert-manager

[cert-manager](https://cert-manager.io/) is used by OpenTelemetry operator.

In this demo, we use the dafault static install:

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml
```

## 2. Install OpenTelemetry operator

[OpenTelemetry operator](https://github.com/open-telemetry/opentelemetry-operator/) is used to inject instrumentation to Java app, and deploy OpenTelemetry collectors.
By default, operator is installed in `opentelemetry-operator-system` namespace.

```
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/download/v0.60.0/opentelemetry-operator.yaml
```

## 3. Install cass-operator

Cassandra cluster is created by [cass-oeprator](https://github.com/k8ssandra/cass-operator/) in the default `cass-operator` namespace.

```
kubectl apply --force-conflicts --server-side -k github.com/k8ssandra/cass-operator/config/deployments/default?ref=v1.12.0
```

## 4. Deploy OpenTelemetry collector

Deploy OpenTelemetry collectors in its own namespace `otel`.

### 4.1. Create `otel` namespace

In this demo, OpenTelemetry collectors that export telemetry to external services are deployed in its own `otel` namespace.

Create namespace:

```
kubectl create namespace otel
```

### 4.2. Create Secrets for external services

Before deploying, generate appropriate secret with API key for external services.

In this demo, [lightstep](https://lightstep.com/), [datadog](https://www.datadoghq.com/), and [Grafana Cloud]() are used.

Create `.env` file according to [.env.sample](./k8s/opentelemetry-collector/.env.sample) file, and create kubernetes Secret from env file.

```
kubectl -n otel create secret generic otel-collector-keys --from-env-file k8s/opentelemetry-collector/.env
```

### 4.3. Deploy OpenTelemetry collector cluster for export

```
kubectl -n otel apply -f k8s/opentelemetry-collector/exporter.yaml
```

This should deploy 3 OpenTelemetry collectors deployment named `exporter`, along with the service `exporter-collector` in `otel` namespace.

## 5. Deploy Cassandra with OpenTelemetry collector

### 5.1. Deploy OpenTelemetry collector sidecar configuration

Cassandra pods are injected with OpenTelemetry collector sidecar by OpenTelemetry operator. Before deploying Cassandra, OpenTelemetry configuration for sidecar should be created in the same namespace.

```
kubectl -n cass-operator apply -f k8s/opentelemetry-collector/otel-sidecar.yaml
```

### 5.2. Create Storage class

Create appropriate storage class named `server-storage` to store Cassandra data if not yet created.

For Amazon EKS, execute:

```
kubectl apply -f https://raw.githubusercontent.com/k8ssandra/cass-operator/master/operator/k8s-flavors/eks/storage.yaml
```

### 5.3. Deploy Cassandra cluster

Deploy Cassandra cluster:

```
kubectl -n cass-operator apply -f k8s/cass-operator/cassdc.yaml
```

## 6. Deploy App

In this demo, the modified version of ["Building an E-commerce Website"](https://github.com/yukim/workshop-ecommerce-app) is used.

The app connects to the deployed Cassandra cluster, and uses OpenTelemetry context propagation for tracing.

### 6.1. Create `app` namespace

In this demo, the demo application is deployed to its own `app` namespace.

Create `app` namespace by executing the following:

```
kubectl create namespace app
```

### 6.2. Deploy OpenTelemetry instrumentation and sidecar configuration

OpenTelemetry project develops auto instrumentation libraries for various programming languages and their frameworks.

[Java instrumentation](https://github.com/open-telemetry/opentelemetry-java-instrumentation/) can automatically instrument the
application that uses Spring framework or DataStax java driver.

OpenTelemetry operator can inject the instrumentation java agent through configuration.

Execute the following to configure instrumentation:

```
kubectl -n app apply -f k8s/app/demo-instrumentation.yaml
```

Also, to deploy OpenTelemetry collector sidecar to the demo application, create sidecar configuration in `app` namespace as well.

```
kubectl -n app apply -f k8s/opentelemetry-collector/otel-sidecar.yaml
```

### 6.3. Prepare the demo application schema and the database user

First, get the super user password from the kubernetes secret:

```
kubectl -n cass-operator get secret otel-demo-superuser --template '{{ .data.password }}' | base64 -d 
```

Use that password to login to the cluster:

```
kubectl -n cass-operator exec -it otel-demo-dc1-rack1-sts-0 -c cassandra -- cqlsh -u otel-demo-superuser
```

Execute the following CQLs to create `ecommerce` keyspace and the database user used from the app:

```
otel-demo-superuser@cqlsh> CREATE KEYSPACE IF NOT EXISTS ecommerce WITH replication = {'class': 'NetworkTopologyStrategy', 'dc1': 3};
otel-demo-superuser@cqlsh> CREATE ROLE demo WITH LOGIN = true AND PASSWORD = 'xxxxxx';
otel-demo-superuser@cqlsh> GRANT ALL PERMISSIONS ON KEYSPACE ecommerce TO demo;
```

While you are in cqlsh, also create tables and insert initial data set according to [this](https://github.com/datastaxdevs/workshop-ecommerce-app#-3b-execute-the-following-cql-script-to-create-the-schema) and [this](https://github.com/datastaxdevs/workshop-ecommerce-app#4-populate-the-data)

### 6.2. Create Secrets for the demo application

The demo application needs to be configured through environmental variables to run.

You need to configure the following in `k8s/app/.env` file:

* Cassandra connection information
* Astra streaming connection
* Google Social login

For Cassandra connection information, put the keyspace name, user name and password you created in the previous step.
For the latter two, please see [the original repository](https://github.com/datastaxdevs/workshop-ecommerce-app) for detail.

Create secret from the env file:

```
kubectl -n app create secret generic demo-app-env --from-env-file k8s/app/.env
```

### 6.3. Deploy app

Execute the following to deploy the demo application:

```
kubectl -n app apply -f k8s/app/deployment.yaml
```

### 6.4. Expose the demo application

Expose the deployed application through load balancer:

```
kubectl -n app expose deployment demo --type=LoadBalancer --port 80 --target-port 8080 --name=demo-lb
```

Wait for the external load balancer to be created.

Make sure to update Google OAuth client setting with the load balancer URL.

## 7. Access the app

Get the URL of the load balancer and access the demo:

```
kubectl -n app get svc
```