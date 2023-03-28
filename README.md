# atomic-fruits-service Project

Simple project exposing and endpoint REST to access Fruit entity.

This project uses Quarkus, the Supersonic Subatomic Java Framework.

If you want to learn more about Quarkus, please visit its website: https://quarkus.io/ .

## Running the application in dev mode

You can run your application in dev mode that enables live coding using:
```shell script
./mvnw compile quarkus:dev
```

> **_NOTE:_**  Quarkus now ships with a Dev UI, which is available in dev mode only at http://localhost:8080/q/dev/.

## Packaging and running the application

The application can be packaged using:
```shell script
./mvnw package
```
It produces the `quarkus-run.jar` file in the `target/quarkus-app/` directory.
Be aware that it’s not an _über-jar_ as the dependencies are copied into the `target/quarkus-app/lib/` directory.

The application is now runnable using `java -jar target/quarkus-app/quarkus-run.jar`.

If you want to build an _über-jar_, execute the following command:
```shell script
./mvnw package -Dquarkus.package.type=uber-jar
```

The application, packaged as an _über-jar_, is now runnable using `java -jar target/*-runner.jar`.

## Creating a native executable

You can create a native executable using: 
```shell script
./mvnw package -Pnative
```

Or, if you don't have GraalVM installed, you can run the native executable build in a container using: 
```shell script
./mvnw package -Pnative -Dquarkus.native.container-build=true
```

You can then execute your native executable with: `./target/atomic-fruits-service-1.0-SNAPSHOT-runner`

If you want to learn more about building native executables, please consult https://quarkus.io/guides/maven-tooling.

Next, we are going to deploy the atomic-fruit-service in a Kubernetes cluster.


# Set up a local Kubernetes environment with KinD

### Install Kind

[kind](https://github.com/kubernetes-sigs/kind) is a tool for running local Kubernetes clusters using Docker container “nodes”.
kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.

Install Kind following the [site instructions](https://kind.sigs.k8s.io/docs/user/quick-start/#installation).

### Create a cluster

Now, you can create a cluster by running the following command:

````shell
bash <(curl -s -L https://raw.githubusercontent.com/snowdrop/k8s-infra/main/kind/kind-reg-ingress.sh)
````

This script creates a Kubernetes cluster using kind tool and deploys a private docker registry and Ingress controller NGINX to route the traffic.

Then, create a namespace to deploy the application and database. You can do it as follows:

````shell
kubectl create namespace grocery
````

## Adding a Data Base to our application
Deploy a PostgreSQL database in the cluster using helm

```shell script
helm install postgresql bitnami/postgresql --version 11.9.1 \
--set auth.database=fruits_database \
--set auth.username=healthy \
--set auth.password=healthy \
--create-namespace -n grocery 

NAME: postgresql
LAST DEPLOYED: Fri Dec  2 10:02:45 2022
NAMESPACE: grocery
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: postgresql
CHART VERSION: 11.9.1
APP VERSION: 14.5.0

** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    postgresql.grocery.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_ADMIN_PASSWORD=$(kubectl get secret --namespace grocery postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

To get the password for "healthy" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace grocery postgresql -o jsonpath="{.data.password}" | base64 -d)

To connect to your database run the following command:

    kubectl run postgresql-client --rm --tty -i --restart='Never' --namespace grocery --image docker.io/bitnami/postgresql:14.5.0-debian-11-r14 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host postgresql -U healthy -d fruits_database -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace grocery svc/postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U healthy -d fruits_database -p 5432

```

Then configure the application to access the database installed. 

There are a few possibilities that are described in the following sections.

## Set up the credentials directly in application.properties file 

````properties
%prod.quarkus.datasource.jdbc.url = jdbc:postgresql://postgresql.grocery:5432/fruits_database
%prod.quarkus.datasource.db-kind = postgresql
%prod.quarkus.datasource.username = healthy
%prod.quarkus.datasource.password = healthy
````

Now you can jump to the [Deploy the application in a Kubernetes cluster](#deploy-the-application-in-a-Kubernetes-cluster) section.

## Getting the database credentials from an existing secret.

### Create and deploy a secret in the cluster with database credentials

We have no secret to keep the database credentials. Let's do something about it. Create a new file `src/main/kubernetes/fruits-database-secret.yml` and paste the following content:

````yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: fruits-database-secret
stringData:
  user: healthy
  password: healthy
````

Deploy the secret in the cluster:

````shell script
kubectl apply -f src/main/kubernetes/fruits-database-secret.yml
````

Now let's add the environment variables to the application.properties file in order to connect to the database
````properties
%prod.quarkus.kubernetes.env.mapping.db-username.from-secret=fruits-database-secret
%prod.quarkus.kubernetes.env.mapping.db-username.with-key=user
%prod.quarkus.kubernetes.env.mapping.db-password.from-secret=fruits-database-secret
%prod.quarkus.kubernetes.env.mapping.db-password.with-key=password
````

Then, replace database related properties with these:

````properties
%prod.quarkus.datasource.username = ${DB_USERNAME}
%prod.quarkus.datasource.password = ${DB_PASSWORD}
````

Now you can jump to the [Deploy the application in a Kubernetes cluster](#deploy-the-application-in-a-Kubernetes-cluster) section.


# Deploy the application in the cluster

We will use a local docker registry and a kind cluster with it enabled.

Customize the container build strategy according to your owns by adding the convenient properties to the `application.properties` file:

```properties
quarkus.container-image.registry=127.0.0.1:5000
quarkus.container-image.image=localhost:5000/amunozhe/atomic-fruits:1.0.0
quarkus.container-image.name=atomic-fruits
quarkus.container-image.group=amunozhe
quarkus.container-image.tag=1.0.0
quarkus.container-image.insecure=true
```

These properties use the local docker registry already mentioned.

Add this couple of properties to `application.properties` so that we trust on the CA cert and expose our application via Ingress.

```properties
quarkus.kubernetes-client.trust-certs=true
quarkus.kubernetes.ingress.expose=true
quarkus.kubernetes.ingress.host=atomic-fruits.127.0.0.1.nip.io
```

Package the application and build and push the container image

```shell script
./mvnw clean package -Dquarkus.container-image.build=true -Dquarkus.container-image.push=true
```

You can check the generated file target/kubernetes/kubernetes.yml, there you'll find: Service, Deployment and Ingress resources.

Finally, deploy the application by running:

````shell script
kubectl apply -f target/kubernetes/kubernetes.yml
````

The other, and equivalent, approach is to use maven command:

````shell script
mvn clean package -Dquarkus.kubernetes.deploy=true
````

If everything went well, you should be able to access the atomic-fruits service using a browser to [http://atomic-fruits.127.0.0.1.nip.io/fruit](http://atomic-fruits.127.0.0.1.nip.io/fruit)

Using Primaza

Install DB following [these instructions](##Adding-a-Data-Base-to-our-application) 
Deploy atomic-fruits **without** datasource configuration by following [this section](#Deploy-the-application-in-the-cluster)
Then, in Primaza:
- Register DB service
- Create credential with healthy/healthy
- Create Claim
- Go to Applications and try to bind the atomic fruits app.

