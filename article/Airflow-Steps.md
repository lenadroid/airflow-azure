# Airflow on Azure

## Architecture

![Diagram for generic use case](tbd)

![Diagram for Azure](tbd)

![Diagram for AWS](tbd)

![Diagram for GCP](tbd)


## Overview

To get started, you will need access to a cloud subscription, such as Azure, AWS, or Google Cloud. The example in this article is based on Azure, however, you should be able to successfully follow the same steps for AWS or GCP with minor changes.

To follow the example on Azure, feel free to create an [account](https://azure.microsoft.com/en-us/free?WT.mc_id=airflow-blog-alehall), and install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest&WT.mc_id=airflow-blog-alehall) or use [Azure Portal](https://azure.microsoft.com/en-us/features/azure-portal?WT.mc_id=airflow-blog-alehall) to create necessary resources. If using Azure CLI, don't forget to login and initialize your session with subscription ID:

```bash
az login
az account set --subscription <subscription-id>
```

To make sure you can easily delete all the resources at once after giving it a try, create an Azure Resource Group that will serve as a grouping unit:

```bash
RESOURCE_GROUP_NAME="airflow"
REGION="East US"
az group create --name $RESOURCE_GROUP_NAME --region $REGION
```

After you are done with resources, feel free to delete the entire resource group:

```bash
az group delete --name $RESOURCE_GROUP_NAME
```

## PostgreSQL

For Apache Airflow, a database is required to store metadata information the status of tasks. Since Airflow is built to work with a metadata database through SQLAlchemy abstraction layer. [SQLAlchemy](https://www.sqlalchemy.org/) is Python SQL toolkit and Object Relational Mapper. Any database that supports SQLAlchemy should work with Airflow. MySQL or PostgreSQL are some of the most common choices.

To create and successfully connect to an instance of PostgreSQL on Azure, please follow detailed instructions [here](https://docs.microsoft.com/en-us/azure/postgresql/quickstart-create-server-database-azure-cli?WT.mc_id=airflow-blog-alehall). 

Feel free to use the same resource group name and location for the PostgreSQL instance. Make sure to indicate a unique server name. I chose `GP_Gen5_2` as an instance size, as 2 vCores and 100 GB of storage is more than enough for the example, but feel free to pick the [size](https://azure.microsoft.com/en-us/pricing/details/postgresql/server?WT.mc_id=airflow-blog-alehall) that fits your own requirements. 

It is important to remember what your PostgreSQL server name, fully qualified domain name, username, and password are. This information will be required during the next steps. Note: you can get fully qualified domain name by looking at the `fullyQualifiedDomainName` after executing the command:

```bash
POSTGRESQL_SERVER_NAME="<your-server-name>"
az postgres server show --resource-group $RESOURCE_GROUP_NAME --name $POSTGRESQL_SERVER_NAME
# example value of fullyQualifiedDomainName on Azure: airflowazuredb.postgres.database.azure.com
```

Check connection to your database using [`PSQL`](https://www.postgresql.org/docs/9.3/app-psql.html):

```bash
P_USER_NAME="your-postgres-username"
psql --host=$POSTGRESQL_SERVER_NAME.postgres.database.azure.com --port=5432 --username=$P_USER_NAME@$POSTGRESQL_SERVER_NAME --dbname=postgres
```

For AWS or GCP, feel free to use Amazon RDS for PostgreSQL or Google Cloud SQL for PostgreSQL respectively.

## File Share

When running data management workflows, we need to store Apache Airflow Directed Acyclic Grapth (DAG) definitions somewhere. When running Apache Airflow locally, we can store them in a local filesystem directory and point to it through the configuration file. When running Airflow in a Docker container (either locally or in the cloud), we have several options.
* Storing data pipeline DAGs directly within the container image. The downside of this approach is when there is a possibility and likelihood of frequent changes to DAGs. This would imply the necessity to rebuild the image each time your DAGs change.
* Storing DAG definitions in a remote Git repository. When there are changes within DAG definitions, using [Git-Sync sidecar](https://github.com/kubernetes/git-sync) can automatically synchronize the repository with the volume in your container.
* Storing DAGs in a shared remote location, such as remote filesystem. Same as with a remote Git repository, we can mount a remote filesystem to a volume in our container and mirror DAG changes automatically. Kubernetes supports a variery of [CSI drivers](https://kubernetes-csi.github.io/docs/drivers.html) for many remote filesystems, including Azure Files, AWS Elastic File System, or Google Cloud Filestore. This approach is great if you also want to store logs somewhere in a remote location.

I am using [Azure Files](https://docs.microsoft.com/en-us/azure/storage/files/storage-how-to-create-file-share?WT.mc_id=airflow-blog-alehall) for storing DAG definitions. To create an Azure fileshare, execute the following commands:

```bash
STORAGE_ACCOUNT="storage-account-name"
FILESHARE_NAME="fileshare-name"

az storage account create \
    --resource-group $RESOURCE_GROUP_NAME \
    --name $STORAGE_ACCOUNT \
    --kind StorageV2 \
    --sku Standard_ZRS \
    --output none

STORAGE_ACCOUNT_KEY=$(az storage account keys list \
    --resource-group $RESOURCE_GROUP_NAME \
    --account-name $STORAGE_ACCOUNT \
    --query "[0].value" | tr -d '"')

az storage share create \
    --account-name $STORAGE_ACCOUNT \
    --account-key $STORAGE_ACCOUNT_KEY \
    --name $FILESHARE_NAME \
    --quota 1024 \
    --output none
```
For Azure Files, detailed creation instructions are located [here](https://docs.microsoft.com/en-us/azure/storage/files/storage-how-to-create-file-share?tabs=azure-cli&WT.mc_id=airflow-blog-alehall). Feel free to create a fileshare on any of the other platforms to follow along.

After the fileshare is created, you can copy one or several DAGs you might have to your newly created storage. If you don't have any DAGs yet, you can use one of those available online. For example, [the following DAG from one of the GitHub repositories](https://github.com/hgrif/airflow-tutorial) called `airflow_tutorial_v01`, which you can also find [here](https://github.com/lenadroid/airflow-azure/blob/master/DAGs/hello_dag.py).

To copy files to Azure Files share, you can use Azure Portal, or [AzCopy](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-files?toc=/azure/storage/files/toc.json#upload-files?WT.mc_id=airflow-blog-alehall) util for programmatic operations.

![Azure Portal Files Upload](images/files-upload.png)

## Kubernetes cluster

For Apache Airflow scheduler, UI, and executor workers, we need to create a cluster. For Azure Kubernetes Service, detailed cluster creation instructions are [here](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal). Make sure to indicate that you'd like the cluster to be provisioned in a Virtual Network (default installation doesn't include it). 

I created the cluster with Azure Portal with the following configuration:

![AKS 1st step](images/aks-1.png)

Node pools:

![AKS 2nd step](images/aks-2.png)

Authentication:

![AKS 3rd step](images/aks-3.png)

Networking:

![AKS 4th step](images/aks-4.png)

Azure Portal has an amazing feature to help you connect to your cluster. On the AKS cluster resource, click `Connect` at the top, and follow the instructions:

![AKS connect](images/aks-connect.png)

## Allow cluster to access database

To make sure the AKS cluster can communicate with the PostgreSQL on Azure database, we need to add a service endpoint on the AKS Virtual Network side and a Virtual Network rule on the PostgreSQL side.

Go to the AKS virtual network resource in Azure Portal, it's located in the same resource group where the cluster is. Under `Service Endpoints` settings menu, select `Add`, and choose `Microsoft.SQL` from the dropdown:

![AKS SQL Service Endpoint](images/aks-sql-endpoint.png)

Go to the PostgreSQL on Azure resource, and under `Connection Security` settings menu `VNET rules` section, select `Add existing virtual network`:

![PostgreSQL VNet Rule 1](images/db-vnet-rule-1.png)

Specify a name for the  Virtual Network rule, select your subscription and the AKS Virtual Network:

![PostgreSQL VNet Rule 2](images/db-vnet-rule-2.png)

These actions will make sure Apache Airflow pods on AKS are able to communicate with the database successfully.

## Prepare fileshare to be used within Kuberneres
Install  a CSI driver corresponding to your platform.

For Azure, follow instructions here to install [Azure File CSI driver](https://github.com/kubernetes-sigs/azurefile-csi-driver/blob/master/docs/install-csi-driver-master.md).

Create a secret to store Azure storage account name and key (make sure `STORAGE_ACCOUNT` and `STORAGE_ACCOUNT_KEY` contain your own values for storage account name and key). This secret will be used later in Persistent Volumes definitions for DAGs and logs.

```bash
kubectl create secret generic azure-secret --from-literal accountname=$STORAGE_ACCOUNT --from-literal accountkey=$STORAGE_ACCOUNT_KEY --type=Opaque
```

## Prepare resource files to run Airflow components on Kubernetes

![resources](images/yamls.png)

You can clone the [GitHub repository](https://github.com/lenadroid/airflow-azure) to get these files. Before you apply them to your own clusters, make sure to review them and read through the notes in this article, as there are quite a few values that need to be customized.

### Namespace

*******

Definition of the namespace is in the [`airflow-namespace.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/airflow-namespace.yaml) file on GitHub.

Create the namespace:
```bash
kubectl create -f airflow-namespace.yaml
```

Make sure to remember to use the right namespace name when creating other resources for Airflow. We will be using `airflow-on-k8s` in this guide.


### Persistent Volumes for logs and DAGs

*******

Definition of a Persistent Volume for DAGs is in [`pv-azurefile-csi.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/pv/pv-azurefile-csi.yaml) file on GitHub.

Definition of a Persistent Volume for logs is in [`pv-azurefile-csi-logs.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/pv/pv-azurefile-csi-logs.yaml) file on GitHub.

Open `pv-azurefile-csi.yaml` and `pv-azurefile-csi-logs.yaml` files to edit them to include your own fileshare name and storage account name. If you followed the steps above, assign the parameter `shareName` to the value of `$FILESHARE_NAME` variable, and parameter `server` to the value of`$STORAGE_ACCOUNT.file.core.windows.net`.

If you are not using Azure, make sure to change the CSI driver settings to correspond to AWS, GCP, or any other platform. Don't forget to create any necessary secrets to store sensitive data corresponding to the fileshare system you are using. 

Create persistent volumes:
```bash
kubectl create -f pv/pv-azurefile-csi.yaml
kubectl create -f pv/pv-azurefile-csi-logs.yaml
```

### Persistent Volume Claims for logs and DAGs

*******

Definition of a Persistent Volume Claim for DAGs is in [`pvc-azurefile-csi-static.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/pv/pvc-azurefile-csi-static.yaml) file on GitHub.

Definition of a Persistent Volume Claim for logs is in [`pvc-azurefile-csi-static-logs.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/pv/pvc-azurefile-csi-static-logs.yaml) file on GitHub.

Create persistent volume claims:
```bash
kubectl create -f pv/pvc-azurefile-csi-static.yaml
kubectl create -f pv/pvc-azurefile-csi-static-logs.yaml
```

After a few minutes, make sure your Persistent Volume Claims are in `Bound` state:

```bash
$ kubectl get pvc -n airflow-on-k8s
NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES
dags-pvc   Bound    dags-pv   10Gi       RWX
logs-pvc   Bound    logs-pv   10Gi       RWX
```

### Service Accounts for scheduler and workers

*******

Definition of Scheduler Service Account is in the [`scheduler-serviceaccount.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/scheduler/scheduler-serviceaccount.yaml) file on GitHub.

Definition of Worker Service Account is in the [`worker-serviceaccount.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/workers/worker-serviceaccount.yaml) file on GitHub.

Create the service accounts:
```bash
kubectl create -f scheduler/scheduler-serviceaccount.yaml
kubectl create -f workers/worker-serviceaccount.yaml
```

### Cluster Role for scheduler and workers to dynamically operate pods

*******

Definition of the Cluster Role is in the [`pod-launcher-role.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/rbac/pod-launcher-role.yaml) file on GitHub.

Create the cluster role:
```bash
kubectl create -f rbac/pod-launcher-role.yaml
```

### Cluster Role Binding for scheduler and workers

*******

Definition of the Cluster Role is in the [`pod-launcher-rolebinding.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/rbac/pod-launcher-rolebinding.yaml) file on GitHub.

Create the cluster role binding:
```bash
kubectl create -f rbac/pod-launcher-rolebinding.yaml
```

### Secrets

#### Fernet key

*******

Start Python shell:
```bash
$ python

Python 3.7.5 (default, Nov  1 2019, 02:16:32)
[Clang 11.0.0 (clang-1100.0.33.8)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

Generate fernet key from Python shell:

```python
from cryptography.fernet import Fernet
fernet_key= Fernet.generate_key()
print('Generated fernet key: ', fernet_key.decode())
```

Example output of this command is:

```bash
Generated fernet key:  ESqYUmi27Udn6wxY83KoM9kuvt9rDcelghHbAgGZW9g=
```

Convert the value to the base64 encoded value:

```bash
$ echo -n "<your-generated-fernet-key-value>" | base64
```

Example output (it will be the value we will use for `fernet-key` within `fernet-secret.yaml` file in this example):

```bash
Slk0OThJbHE4R0xRNEFuRlJWT3FUR3lBeDg3bG5BWWhEdWx1ekhHX2RJQT0=
```

Definition of Fernet Key secret is in the [`fernet-secret.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/secrets/fernet-secret.yaml) file on GitHub.

Replace the value of the `fernet-key` parameter in the file with your generated fernet-key value.

Create the fernet secret:

```bash
kubectl create -f secrets/fernet-secret.yaml
```

#### Database connection information

*******

Prepare your PostgreSQL database connection. Generally, it follows the format of:

```bash
postgresql+psycopg2://user:password@hostname:5432/dbname
```

> Note: when using Azure SQL Database for PostgreSQL, the connection string requires user to be in format of `user@host`, where @ sign should be escaped as %40 (more [details](https://github.com/MicrosoftDocs/azure-docs/issues/6371)):
> `postgresql+psycopg2://user%40host:password@host.postgres.database.azure.com:5432/dbname`

Encode your connection using base64 (replace `airflowusername` with your username, `airflowpassword` with your password, `airflowhost` with your host, and `airflow` with your database name):

```bash
echo -n "postgresql+psycopg2://airflowuser%40airflowhost:airlowpassword@airflowhost.postgres.database.azure.com:5432/airflow" | base64
```

Example outout (this will be the value we will use for `connection` within `metadata-connection-secret.yaml` file in this example):

```bash
cG9zdGdyZXNxbCtwc3ljb3BnMjovL2xlbmElNDBhaXJmbG93YXp1cmVkYjpQYXNzd29yZDEhQGFpcmZsb3dhenVyZWRiLnBvc3RncmVzLmRhdGFiYXNlLmF6dXJlLmNvbTo1NDMyL3Bvc3RncmVz
```

Definition of Fernet Key secret is in the [`metadata-connection-secret.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/secrets/metadata-connection-secret.yaml) file on GitHub.

Replace the value of the `connection` parameter in the file with your base64-encoded connection value.

Create the Airflow connection metadata secret:

```bash
kubectl create -f secrets/metadata-connection-secret.yaml
```

### Config Map

*******

Definition of Config Map is in the [`configmap.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/configmap.yaml) file on GitHub.

If you are following this guide for Airflow with 

dags_folder - Ensure the right folder is used for DAGs
dags_in_image - If set to False, it will look at mounted volumes or git repo
dags_volume_claim
dags_volume_mount_point
git_
worker_service_account_name

Create config map:

```bash
kubectl create -f configmap.yaml
```

### StatsD

*******

Definition of a StatsD deployment is in [`statsd-deployment.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/statsd/statsd-deployment.yaml) file on GitHub.

Definition of a StatsD service is in [`statsd-service.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/statsd/statsd-service.yaml) file on GitHub.

Check the status of the StatsD instance and get its TCP port:

```bash
kubectl get services -n airflow-on-k8s
```

```bash
NAME        TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
statsd      ClusterIP      10.0.9.199    <none>        9125/UDP,9102/TCP   24h
```

Map the StatsD instance to local port (for example 7000):
```bash
kubectl port-forward service/statsd 7000:9102 -n airflow-on-k8s
```

Output:

```bash
Forwarding from 127.0.0.1:7000 -> 9102
Forwarding from [::1]:7000 -> 9102
```

Open the `127.0.0.1:7000` page in your browser to see StatsD page:

![statsd open page](images/statsd-1.png)

![statsd metrics page](images/statsd-2.png)


### Scheduler

*******

Definition of the Scheduler deployment is in [`scheduler-deployment.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/scheduler/scheduler-deployment.yaml) file on GitHub.

Create scheduler deployment:

```bash
kubectl create -f scheduler/scheduler-deployment.yaml
```

### Webserver

*******

Definition of Webserver deployment is in [`webserver-deployment.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/webserver/webserver-deployment.yaml) file on GitHub.

Definition of Webserver service is in [`webserver-service.yaml`](https://github.com/lenadroid/airflow-azure/blob/master/resource-definitions/webserver/webserver-service.yaml) file on GitHub.

If you'd like the Webserver to have an external IP, replace `ClusterIP` with `LoadBalancer` in the `webserver-service.yaml`, and you will be able to access from the outside of the cluster without proxies or port forwarding.

Create webserver deployment and service:

```bash
kubectl create -f webserver/webserver-deployment.yaml
kubectl create -f webserver/webserver-service.yaml
```

Check the status of the Airflow UI instance and get its TCP port:

```bash
kubectl get services -n airflow-on-k8s
```

```bash
NAME        TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
statsd      ClusterIP      10.0.9.199    <none>        9125/UDP,9102/TCP   24h
webserver   ClusterIP      10.0.9.175    <none>        8080:31003/TCP      19h
```

Map the Airflow UI instance to local port (for example 8080):
```bash
kubectl port-forward service/webserver 8080:8080 -n airflow-on-k8s
```

Output:

```bash
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

Open the `127.0.0.1:8080` page in your browser to see AirflowUI page:

![Airflow UI](images/airflow-ui.png)

If you'd like to create a new user for Airflow Webserver, you can connect to the webserver pod:

```bash
$ kubectl exec --stdin --tty webserver-647fdcb7c-4qkx9 -n airflow-on-k8s -- /bin/bash
airflow@webserver-647fdcb7c-4qkx9:/opt/airflow$
```

And create a user from Aiflow CLI. Replace `USERNAME`, `PASSWORD`, `FIRSTNAME`, `LASTNAME`, `EMAIL`, `ROLE` with your own values. Note: existing Airflow roles can be one of the follwing - `Admin`, `User`, `Op`, `Viewer`, and `Public`:

```bash
airflow create_user -u USERNAME -p PASSWORD -f FIRSTNAME -l LASTNAME -e EMAIL -r ROLE
```

Example output:

```bash
[2020-08-08 00:00:40,140] {__init__.py:51} INFO - Using executor KubernetesExecutor
[2020-08-08 00:00:40,143] {dagbag.py:396} INFO - Filling up the DagBag from /opt/airflow/dags
[2020-08-08 00:00:41,834] {security.py:475} INFO - Start syncing user roles.
[2020-08-08 00:00:42,458] {security.py:385} INFO - Fetching a set of all permission, view_menu from FAB meta-table
[2020-08-08 00:00:42,833] {security.py:328} INFO - Cleaning faulty perms
Viewer user newuser created.
```

Afterwards, you can login to Airlow UI with credentials of any of the users you have provisioned.

Airflow DAGs:

![Airflow UI](images/airflow-dags.png)

Airflow DAG graph:

![Airflow UI](images/airflow-dag-graph.png)

Airflow DAG runs:

![Airflow UI](images/airflow-dag-runs.png)

Airflow DAG logs:

![Airflow UI](images/airflow-dag-logs.png)


### Checking health of resources provisioned

Check if scheduler, webserver, and statsd deployments are in a healthy state:

```bash
$ kubectl get deployments -n airflow-on-k8s
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
scheduler   1/1     1            1           123m
statsd      1/1     1            1           24h
webserver   1/1     1            1           122m
```

Check if all corresponding pods are healthy:

```bash
$ kubectl get po -n airflow-on-k8s
NAME                         READY   STATUS    RESTARTS   AGE
scheduler-7584f4b4b7-5zhvw   2/2     Running   0          125m
statsd-d6d5bcd7c-dg26n       1/1     Running   0          24h
webserver-647fdcb7c-4qkx9    1/1     Running   0          124m
```

Check status/events of any particular pod (`scheduler-7584f4b4b7-5zhvw` pod in this example):

```bash
$ kubectl describe pod scheduler-7584f4b4b7-5zhvw -n airflow-on-k8s
```

Check pod logs (where `-c` parameter refers to the name of init-container we want to check on, `scheduler` in this case):

```bash
kubectl logs scheduler-7584f4b4b7-5zhvw -n airflow-on-k8s -c scheduler
```

Check logs from init-container of a pod (where `-c` parameter refers to the name of init-container we want to check on, `run-airflow-migrations` in this case):

```bash
$ kubectl logs scheduler-7584f4b4b7-5zhvw -n airflow-on-k8s -c run-airflow-migrations
```
Connect to a pod to execute commands from within (`webserver-647fdcb7c-4qkx9` pod in this example):

```bash
kubectl exec --stdin --tty webserver-647fdcb7c-4qkx9 -n airflow-on-k8s -- /bin/bash
```

After getting connected, we can execute commands (for example check the `dags` directory):

```bash
airflow@webserver-647fdcb7c-4qkx9:/opt/airflow$ cd dags
airflow@webserver-647fdcb7c-4qkx9:/opt/airflow/dags$ ls
__pycache__  dag_processor_manager  dags  first-dag.py  hello_dag.py  hello_world.py  outfile  scheduler
```

### Overview of resources created

We can see which resources are running in the cluster by running the following command:

```bash
kubectl get all -n airflow-on-k8s
```

```bash
NAME                             READY   STATUS    RESTARTS   AGE
pod/scheduler-7584f4b4b7-jdfzl   2/2     Running   0          13m
pod/statsd-d6d5bcd7c-mjdm7       1/1     Running   0          17m
pod/webserver-647fdcb7c-ft72t    1/1     Running   0          8m26s

NAME                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
service/statsd      ClusterIP   10.0.47.229   <none>        9125/UDP,9102/TCP   16m
service/webserver   ClusterIP   10.0.197.80   <none>        8080/TCP            6s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/scheduler   1/1     1            1           13m
deployment.apps/statsd      1/1     1            1           17m
deployment.apps/webserver   1/1     1            1           8m26s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/scheduler-7584f4b4b7   1         1         1       13m
replicaset.apps/statsd-d6d5bcd7c       1         1         1       17m
replicaset.apps/webserver-647fdcb7c    1         1         1       8m26s
```

Note: it doesn't show the secrets, persistent volumes, persistent volume claims, service accounts, cluster roles, or cluster role bindings that we also created.

In Azure Portal, you can see all the resources within the main resource group:

![Resource Group resources](images/azure-resources.png)

To clean up your environment, just run:

```bash
az group delete --name $RESOURCE_GROUP_NAME
```