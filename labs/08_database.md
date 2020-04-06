# Lab 8: Attaching a Database

Numerous applications are in some kind stateful and want to save data persistently, be it in a database or as files on a filesystem or in an object store. During this lab we are going to create a MySQL service and attach it to our application so that application pods can access the same database.


## Task: LAB8.1: Create the MySQL Service


We are first going to create a so-called secret in which we write the password for accessing the database.

```bash
$ kubectl create secret generic mysql-password --from-literal=password=mysqlpassword --namespace [USER]
secret/mysql-password created
```

The secret will neither be shown with `kubectl get` nor with `kubectl describe`:

```bash
$ kubectl get secret mysql-password --namespace [USER] -o json
{
    "apiVersion": "v1",
    "data": {
        "password": "bXlzcWxwYXNzd29yZA=="
    },
    "kind": "Secret",
    "metadata": {
        "creationTimestamp": "2018-10-16T13:36:15Z",
        "name": "mysql-password",
        "namespace": "[namespace]",
        "resourceVersion": "3156527",
        "selfLink": "/api/v1/namespaces/[namespace]/secrets/mysql-password",
        "uid": "74a7f030-d148-11e8-a406-42010a840034"
    },
    "type": "Opaque"
}
```

The string at `.data.password` is base64 encoded and can easily be decoded:

```bash
$ echo "bXlzcWxwYXNzd29yZA=="| base64 -d
mysqlpassword
```

**Note:** Secrets by default are not encrypted! Kubernetes 1.13 [offers this capability](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/). Another option would be the use of a vault like [Vault by HashiCorp](https://www.vaultproject.io/).

We are going to create another secret for storing the MySQL root password.

```bash
$ kubectl create secret generic mysql-root-password --from-literal=password=mysqlrootpassword --namespace [USER] 
secret/mysql-root-password created
```

We are now going to create deployment and service. As a first example we use a database without persistent storage. Only use an ephemeral database for testing purposes as a restart of the pod leads to the loss of all saved data. We are going to look at how to persist this data in a persistent volume later on.

As we had seen in the earlier labs, all resources like deployments, services, secrets and so on can be displayed in yaml or json format. But it doesn't end there, capabilities also include the creation and exportation of resources using yaml or json files.

In our case we want to create a deployment including a service for our MySQL database:


```
$ kubectl create -f ./labs/08_data/mysql-deployment-empty.yaml --namespace [USER]
service/springboot-mysql created
deployment.apps/springboot-mysql created
```

As soon as the container image for mysql:5.6 has been pulled, you will see a new pod using `kubectl get pods`.

The environment variables defined in the deployment configure the MySQL pod and how our frontend will be able to access it.


## Task: LAB8.2: Attaching the Database to the Application

By default our example-spring-boot application uses a H2 memory database. However, this can be changed by defining the following environment variables to use the newly created MySQL service:

- SPRING_DATASOURCE_USERNAME springboot
- SPRING_DATASOURCE_PASSWORD [value from the secret]
- SPRING_DATASOURCE_DRIVER_CLASS_NAME com.mysql.jdbc.Driver
- SPRING_DATASOURCE_URL jdbc:mysql://[MySQL service address]/springboot?autoReconnect=true

You can either use the MySQL service's cluster ip or DNS name as address. All services and pods can be resolved by DNS using their name.

For us this means we can use the following value as `SPRING_DATASOURCE_URL`:


```bash
Name des Services: springboot-mysql

jdbc:mysql://springboot-mysql/springboot?autoReconnect=true
```

We now can set these environment variables inside the deployment configuration. The configuration change automatically triggers a new deployment of the application. Because we set the environment variables the application now tries to connect to the MySQL database and [Liquibase](http://www.liquibase.org/) creates the schema and imports some test data.

**Note:** Liquibase is an open source, database-independent library for managing and applying database changes. Liquibase recognizes at application startup if database changes are necessary. You can also look at the logs to find out if Liquibase did some changes or not.

So let's set the environment variables in the example-spring-boot deployment:

```
$ kubectl set env deployment/example-spring-boot SPRING_DATASOURCE_USERNAME=springboot SPRING_DATASOURCE_PASSWORD=mysqlpassword SPRING_DATASOURCE_DRIVER_CLASS_NAME="com.mysql.jdbc.Driver" SPRING_DATASOURCE_URL="jdbc:mysql://springboot-mysql/springboot?autoReconnect=true" --namespace [USER]
```

You could also do the changes by direclty editing the deployment:

```
$ kubectl edit deployment --namespace [USER] example-spring-boot
```

```
$ kubectl get deployment --namespace [USER] example-spring-boot
```
```
...
      - env:
        - name: SPRING_DATASOURCE_USERNAME
          value: springboot
        - name: SPRING_DATASOURCE_PASSWORD
          value: mysqlpassword
        - name: SPRING_DATASOURCE_DRIVER_CLASS_NAME
          value: com.mysql.jdbc.Driver
        - name: SPRING_DATASOURCE_URL
          value: jdbc:mysql://springboot-mysql/springboot?autoReconnect=true

...
```

In order to find out if the change worked we can either look at the springboot container's logs (**Tip**: `kubectl logs [POD NAME]`).
Or we could register some "Hellos" in the application, delete the pod, wait for the new pod to be started and check if they are still there.

**Attention:** This does not work if we delete the database pod as its data is not yet persisted.


## Task: LAB8.3: Manual Database Connection

As described in [lab 07](07_troubleshooting_ops.md) we can log into a pod with `kubectl exec -it [POD NAME] -- /bin/bash`.

Show all pods:

```
$ kubectl get pods --namespace [USER]
NAME                                   READY   STATUS    RESTARTS   AGE
example-spring-boot-574544fd68-qfkcm   1/1     Running   0          2m20s
springboot-mysql-f845ccdb7-hf2x5       1/1     Running   0          31m
```

Log into the MySQL pod:

```
$ kubectl exec -it springboot-mysql-f845ccdb7-hf2x5 --namespace [USER] -- /bin/bash
```

You are now able to connect to the database and display the tables. Log in using:

```
$ mysql -u$MYSQL_USER -p$MYSQL_PASSWORD springboot
Warning: Using a password on the command line interface can be insecure.
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 12
Server version: 5.6.41 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```


Show all tables with:
```
show tables;
```


## Task: LAB8.4: Import a Database Dump

Our task is now to import this [dump](https://raw.githubusercontent.com/appuio/techlab/lab-3.3/labs/data/08_dump/dump.sql) into the MySQL database running as a pod. Use the `mysql` command line utility to do this. Make sure the database is empty beforehand. You could also delete and recreate the database.

**Tip:** You can also copy local files into a pod using `kubectl cp`. Be aware that the `tar` binary has to be present inside the container and on your operating system in order for this to work! Install `tar` on UNIX systems with e.g. your package manager, on Windows there's e.g. [cwRsync](https://www.itefix.net/cwrsync). If you cannot install `tar` on your host, there's also the possibility of logging into the pod and using `curl -O [URL]`.


## Task: LAB8.5: Springboot Deployment Password from Secret

Analogue to the `MYSQL_PASSWORD` environment variable, remove the value of the `SPRING_DATASOURCE_PASSWORD` environment variable and instead insert a reference to the secret which contains the password.


---

## Solution: LAB8.4

This is how you copy the database dump into the pod:

```
kubectl cp ./labs/08_data/dump/ springboot-mysql-f845ccdb7-hf2x5:/tmp/ --namespace [USER]
```

This is how you log into the MySQL pod:

```
$ kubectl exec -it springboot-mysql-f845ccdb7-hf2x5 --namespace [USER] -- /bin/bash
```

This shows how to drop the whole database:
```
$ mysql -u$MYSQL_USER -p$MYSQL_PASSWORD  springboot
...
mysql> drop database springboot;
mysql> create database springboot;
mysql> exit
```
Importing a dump:

```
$ mysql -u$MYSQL_USER -p$MYSQL_PASSWORD springboot < /tmp/dump/dump.sql
```

**Note:** A database dump can be created as follows:

```
mysqldump --user=$MYSQL_USER --password=$MYSQL_PASSWORD springboot > /tmp/dump.sql
```


---
**End of lab 8**

<p width="100px" align="right"><a href="09_persistent_storage.md">Persistent Storage →</a></p>

[← back to the overview](../README.md)
