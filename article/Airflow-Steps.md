# Airflow on Azure

## Overview

## PostgreSQL

## File Share

Copy one or several DAGs you might have to your newly created fileshare. If you don't have any DAGs yet, you can use one of those available online. For example, [the following DAG from one of the GitHub repositories](https://github.com/hgrif/airflow-tutorial) called `airflow_tutorial_v01`, which you can also find [here](todo).

## Kubernetes cluster

## Allow cluster to access database

## Prepare fileshare to be used within Kuberneres

Create a secret to store Azure storage account name and key (replace `NAME` and `KEY` with your own values for storage account name). This secret will be used later in Persistent Volumes definitions for DAGs and logs.

```bash
$ kubectl create secret generic azure-secret --from-literal accountname=NAME --from-literal accountkey="KEY" --type=Opaque
```

You can get the `NAME` and `KEY` values from your Azure storage account from Azure Portal:

![Storage Account Keys](images/storage-account-keys-portal.png)

Alternatively, with Azure CLI (replace `NAME` with your own storage account name):

```bash
$ az storage account keys list --account-name NAME
```

Output:
```bash
[
  {
    "keyName": "key1",
    "permissions": "Full",
    "value": "******"
  },
  {
    "keyName": "key2",
    "permissions": "Full",
    "value": "******"
  }
]
```

## Prepare resource files to run Airflow components on Kubernetes

![resources](images/yamls.png)

You can clone the [GitHub repository]() to get these files. Before you apply them to your own clusters, make sure to review them and read through the notes in this article, as there are quite a few values that need to be customized.

### Namespace

Definition of the namespace in the `airflow-namespace.yaml` file:

```yaml
kind: Namespace
apiVersion: v1
metadata:
  # Replace with your own namespace name, if you'd like
  name: airflow-on-k8s
```

Create the namespace:
```bash
$ kubectl create -f airflow-namespace.yaml
```

Make sure to remember to use the right namespace name when creating other resources for Airflow. We will be using `airflow-on-k8s` in this guide.


### Persistent Volumes for logs and DAGs

Definition of a Persistent Volume for DAGs in `pv-azurefile-csi.yaml` file (in this example, `airfilesystem` Azure file share contains DAG definitions):

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  # Namespace name isn't necessary for Persistent Volumes
  # Take note of the Persistent Volume name for DAGs, as it is used in other templates
  name: dags-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain  # if set as "Delete" file share would be removed in pvc deletion
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
    - uid=0
    - gid=0
    - mfsymlinks
    - cache=strict  # https://linux.die.net/man/8/mount.cifs
    - nosharesock
  csi:
    # Azure CSI driver settings for an existing fileshare
    driver: file.csi.azure.com
    readOnly: false
    volumeHandle: fs-dags  # make sure it's a unique id in the cluster
    volumeAttributes:
      # Replace shareName and server with values corresponding to your own fileshare
      shareName: airfilesystem  # only Azure file share name, don't use full path
      server: airfilesystem.file.core.windows.net  # optional, provide a new address to replace default "accountname.file.core.windows.net"
    nodeStageSecretRef:
      name: azure-secret
      namespace: default
```

Definition of a Persistent Volume for logs in `pv-azurefile-csi-logs.yaml` file (in this example, `airfilesystem` Azure file share contains DAG definitions):
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  # Namespace name isn't necessary for Persistent Volumes
  # Take note of the Persistent Volume name for logs, as it is used in other templates
  name: logs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain  # if set as "Delete" file share would be removed in pvc deletion
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
    - uid=0
    - gid=0
    - mfsymlinks
    - cache=strict  # https://linux.die.net/man/8/mount.cifs
    - nosharesock
  csi:
    # Azure CSI driver settings for an existing fileshare
    driver: file.csi.azure.com
    readOnly: false
    volumeHandle: fs-logs  # make sure it's a unique id in the cluster
    volumeAttributes:
      # Replace shareName and server with values corresponding to your own fileshare
      shareName: airfilesystem  # only file share name, don't use full path
      server: airfilesystem.file.core.windows.net  # optional, provide a new address to replace default "accountname.file.core.windows.net"
    nodeStageSecretRef:
      name: azure-secret
      namespace: default
```

Create persistent volumes:
```bash
$ kubectl create -f pv/pv-azurefile-csi.yaml
$ kubectl create -f pv/pv-azurefile-csi-logs.yaml
```

### Persistent Volume Claims for logs and DAGs

Definition of a Persistent Volume Claim for DAGs in `pvc-azurefile-csi-static.yaml` file:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  # Take note of the Persistent Volume Claim name for DAGs, as it is used in other templates
  name: dags-pvc
  # Make sure to specify the right namespace name
  namespace: airflow-on-k8s
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  # Refers to the DAGs Persistent Volume created earlier
  volumeName: dags-pv
  storageClassName: ""
```

Definition of a Persistent Volume Claim for logs in `pvc-azurefile-csi-static-logs.yaml` file:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  # Take note of the Persistent Volume Claim name for logs, as it is used in other templates
  name: logs-pvc
  # Make sure to specify the right namespace name
  namespace: airflow-on-k8s
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  # Refers to the logs Persistent Volume created earlier
  volumeName: logs-pv
  storageClassName: ""
```

Create persistent volume claims:
```bash
$ kubectl create -f pv/pvc-azurefile-csi-static.yaml
$ kubectl create -f pv/pvc-azurefile-csi-static-logs.yaml
```

After a few minutes, make sure your Persistent Volume Claims are in `Bound` state:

```bash
$ kubectl get pvc -n airflow-on-k8s
NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES
dags-pvc   Bound    dags-pv   10Gi       RWX
logs-pvc   Bound    logs-pv   10Gi       RWX
```

### Service Accounts for scheduler and workers

Definition of Scheduler Service Account in the `scheduler-serviceaccount.yaml` file:

```yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: scheduler-serviceaccount
  # Make sure to specify the right namespace name
  namespace: airflow-on-k8s
  labels:
    tier: airflow
    release: RELEASE-NAME
    chart: "airflow-1.0.0"
    heritage: Helm
```

Definition of Worker Service Account in the `worker-serviceaccount.yaml` file:

```yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: worker-serviceaccount
  # Make sure to specify the right namespace name
  namespace: airflow-on-k8s
  labels:
    tier: airflow
    release: RELEASE-NAME
    chart: "airflow-1.0.0"
    heritage: Helm
```

Create the service accounts:
```bash
$ kubectl create -f scheduler/scheduler-serviceaccount.yaml
$ kubectl create -f workers/worker-serviceaccount.yaml
```

### Cluster Role for scheduler and workers to dynamically operate pods

Definition of the Cluster Role in the `pod-launcher-role.yaml` file:

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # No namespace indication required
  name: pod-launcher-role
rules:
  - apiGroups:
      - ""
    resources:
      - "pods"
    verbs:
      - "create"
      - "list"
      - "get"
      - "watch"
      - "delete"
  - apiGroups:
      - ""
    resources:
      - "pods/log"
    verbs:
      - "get"
  - apiGroups:
      - ""
    resources:
      - "pods/exec"
    verbs:
      - "create"
      - "get"

```

Create the cluster role:
```bash
$ kubectl create -f rbac/pod-launcher-role.yaml
```

### Cluster Role Binding for scheduler and workers

Definition of the Cluster Role in the `pod-launcher-rolebinding.yaml` file:

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # No namespace indication required
  name: pod-launcher-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-launcher-role
subjects:
    # Scheduler service account
  - kind: ServiceAccount
    name: scheduler-serviceaccount
    namespace: airflow-on-k8s
    # Workers service account
  - kind: ServiceAccount
    name: worker-serviceaccount
    namespace: airflow-on-k8s
```

Create the cluster role binding:
```bash
$ kubectl create -f rbac/pod-launcher-rolebinding.yaml
```

### Secrets

#### Fernet key

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
RVNxWVVtaTI3VWRuNnd4WTgzS29NOWt1dnQ5ckRjZWxnaEhiQWdHWlc5Zz0=
```

Definition of Fernet Key secret in the `fernet-secret.yaml` file:

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: fernet-key
  # Make sure to specify the right namespace name
  namespace: airflow-on-k8s
  labels:
    release: RELEASE-NAME
    chart: airflow
    heritage: Helm
type: Opaque
data:
  # Base64 encoded value of the fernet key
  # Replace with your own generated base64 encoded value
  fernet-key: "b19yeDlYOHh4OWVhLVhESlRPSll1cXVJOWdyZzJIUzRBcjZCampEVnlEUT0="
```

Create the fernet secret:

```bash
kubectl create -f secrets/fernet-secret.yaml
```

#### Database connection information

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
cG9zdGdyZXNxbCtwc3ljb3BnMjovL2FpcmZsb3d1c2VyJTQwYWlyZmxvd2hvc3Q6YWlybG93cGFzc3dvcmRAYWlyZmxvd2hvc3QucG9zdGdyZXMuZGF0YWJhc2UuYXp1cmUuY29tOjU0MzIvYWlyZmxvdw==
```


Definition of Fernet Key secret in the `metadata-connection-secret.yaml` file:

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: airflow-metadata
  # Make sure to specify the right namespace name
  namespace: airflow-on-k8s
  labels:
    release: RELEASE-NAME
    chart: airflow
    heritage: Helm
type: Opaque
data:
  # Base64 encoded value of the database connection
  # Replace with your own generated base64 encoded value

  connection: "cG9zdGdyZXNxbCtwc3ljb3BnMjovL2FpcmZsb3clNDBhaXJwb3N0Z3JlczphaXJmbG93QGFpcnBvc3RncmVzLnBvc3RncmVzLmRhdGFiYXNlLmF6dXJlLmNvbTo1NDMyL2FpcmZsb3c="
```

Create the fernet secret:

```bash
kubectl create -f secrets/metadata-connection-secret.yaml
```

### Config Map

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: airflow-config
  # Make sure to specify the right namespace name
  namespace: airflow-on-k8s
  labels:
    tier: airflow
    component: config
    release: RELEASE-NAME
    chart: "airflow-1.0.0"
    heritage: Helm
data:
  # These are system-specified config overrides.
  airflow.cfg: |-
    [api]
    auth_backend = airflow.api.auth.backend.deny_all
    
    [celery]
    default_queue = celery
    
    [core]
    colored_console_log = False
# Ensure the right folder is used for DAGs
    dags_folder = /opt/airflow/dags
    executor = KubernetesExecutor
    load_examples = False
    remote_logging = False
    
    [elasticsearch]
    json_format = True
    log_id_template = {dag_id}_{task_id}_{execution_date}_{try_number}
    
    [elasticsearch_configs]
    max_retries = 3
    retry_timeout = True
    timeout = 30
    
    [kubernetes]
    airflow_configmap = airflow-config
    airflow_local_settings_configmap = airflow-config
# If set to False, it will look at mounted volumes or git repo
    dags_in_image = False
# Ensure the right Persistent Volume Claim name and mount point for DAGs
    dags_volume_claim = dags-pvc
    dags_volume_mount_point = /opt/airflow/dags
    dags_volume_subpath = 
    delete_worker_pods = True
    fs_group = 50000
    git_branch = v1-10-stable
    git_dags_folder_mount_point = 
    git_repo = 
    git_ssh_key_secret_name = 
    git_ssh_known_hosts_configmap_name = 
    git_sync_container_repository = k8s.gcr.io/git-sync
    git_sync_container_tag = v3.1.6
    git_sync_credentials_secret = 
    git_sync_depth = 1
    git_sync_dest = repo
    git_sync_init_container_name = git-sync
    git_sync_rev = HEAD
    git_sync_root = /git
    git_sync_run_as_user = 50000
    image_pull_secrets = registry
# Make sure to specify the right namespace name
    namespace = airflow-on-k8s
    run_as_user = 50000
    worker_container_image_pull_policy = IfNotPresent
    worker_container_repository = apache/airflow
    worker_container_tag = 1.10.10.1-alpha2-python3.6
# Make sure to specify the right worker service account name
    worker_service_account_name = worker-serviceaccount
    
    [kubernetes_secrets]
# Make sure to specify the right secret names
    AIRFLOW__CORE__FERNET_KEY = fernet-key=fernet-key
    AIRFLOW__CORE__SQL_ALCHEMY_CONN = airflow-metadata=connection
    
    [logging]
    logging_level = DEBUG
    
    [scheduler]
    run_duration = 41460
    scheduler_heartbeat_sec = 5
    statsd_host = statsd
    statsd_on = True
    statsd_port = 9125
    statsd_prefix = airflow
    
    [webserver]
# Make sure to enable password authentication with webserver
    enable_proxy_fix = True
    expose_config = True
    rbac = True
    authenticate = True
    auth_backend = airflow.contrib.auth.backends.password_auth

```

Create config map:

```bash
kubectl create -f configmap.yaml
```


### Scheduler

Definition of the Scheduler deployment in `scheduler-deployment.yaml`:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: scheduler
  # Make sure to specify the right namespace name
  namespace: airflow-on-k8s
  labels:
    tier: airflow
    component: scheduler
    release: RELEASE-NAME
    chart: "airflow-1.0.0"
    heritage: Helm
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: airflow
      component: scheduler
      release: RELEASE-NAME
  template:
    metadata:
      labels:
        tier: airflow
        component: scheduler
        release: RELEASE-NAME
      annotations:
        checksum/metadata-secret: a3512f27fea8455cdddc51ef650052d74657bcaa16194d24b555417e312d43da
        checksum/result-backend-secret: 4bd4a60ef60435fe29fc8135a43a436c0854074a228246c67a6e7488b138200f
        checksum/pgbouncer-config-secret: da52bd1edfe820f0ddfacdebb20a4cc6407d296ee45bcb500a6407e2261a5ba2
        checksum/airflow-config: 6132d4c762bec566a83667e8a23486fcbc29157811f277b66e6568047f627c14
        cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
    spec:
      nodeSelector:
        {}
      affinity:
        {}
      tolerations:
        []
      restartPolicy: Always
      terminationGracePeriodSeconds: 10
      # Make sure to indicate the service account that has rights to perform operations with pods
      serviceAccountName: scheduler-serviceaccount
      securityContext:
        runAsUser: 50000
        fsGroup: 50000
      initContainers:
        - name: run-airflow-migrations
          image: apache/airflow
          imagePullPolicy: IfNotPresent
          args: ["bash", "-c", "airflow upgradedb || airflow db upgrade"]
          env:    
          # Getting values from the secrets     
            - name: AIRFLOW__CORE__FERNET_KEY
              valueFrom:
                secretKeyRef:
                  name: fernet-key
                  key: fernet-key
            - name: AIRFLOW__CORE__SQL_ALCHEMY_CONN
              valueFrom:
                secretKeyRef:
                  name: airflow-metadata
                  key: connection
            - name: AIRFLOW_CONN_AIRFLOW_DB
              valueFrom:
                secretKeyRef:
                  name: airflow-metadata
                  key: connection
      containers:
        - name: scheduler
          image: apache/airflow
          imagePullPolicy: IfNotPresent
          args: ["scheduler"]
          env:
          # Getting values from the secrets          
            - name: AIRFLOW__CORE__FERNET_KEY
              valueFrom:
                secretKeyRef:
                  name: fernet-key
                  key: fernet-key
            - name: AIRFLOW__CORE__SQL_ALCHEMY_CONN
              valueFrom:
                secretKeyRef:
                  name: airflow-metadata
                  key: connection
            - name: AIRFLOW_CONN_AIRFLOW_DB
              valueFrom:
                secretKeyRef:
                  name: airflow-metadata
                  key: connection
          livenessProbe:
            failureThreshold: 10
            periodSeconds: 30
            exec:
              command:
              - python
              - -Wignore
              - -c
              - |
                import os
                os.environ['AIRFLOW__CORE__LOGGING_LEVEL'] = 'ERROR'
                os.environ['AIRFLOW__LOGGING__LOGGING_LEVEL'] = 'ERROR'

                from airflow.jobs.scheduler_job import SchedulerJob
                from airflow.utils.net import get_hostname
                import sys

                job = SchedulerJob.most_recent_job()
                sys.exit(0 if job.is_alive() and job.hostname == get_hostname() else 1)
          resources:
            {}
          volumeMounts:
          # Mounting config, logs, and dags directories to scheduler pods
            - name: logs-pv
              mountPath: "/opt/airflow/logs"
            - name: dags-pv
              mountPath: "/opt/airflow/dags"
            - name: config
              mountPath: "/opt/airflow/airflow.cfg"
              subPath: airflow.cfg
              readOnly: true
        - name: scheduler-gc
          image: apache/airflow
          imagePullPolicy: IfNotPresent
          args: ["bash", "/clean-logs"]
          volumeMounts:
            - name: logs-pv
              mountPath: "/opt/airflow/logs"
      volumes:
      # Specifying volume information for config, logs, and dags
        - name: config
          configMap:
            name: airflow-config
        - name: dags-pv
          persistentVolumeClaim:
            claimName: dags-pvc
        - name: logs-pv
          persistentVolumeClaim:
            claimName: logs-pvc
```

Create scheduler deployment:

```bash
kubectl create -f scheduler-deployment.yaml
```

### Webserver

Definition of Webserver deployment in `webserver-deployment.yaml`:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: webserver
  # Make sure to specify the right namespace name
  namespace: airflow-on-k8s
  labels:
    tier: airflow
    component: webserver
    release: RELEASE-NAME
    chart: "airflow-1.0.0"
    heritage: Helm
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      tier: airflow
      component: webserver
      release: RELEASE-NAME
  template:
    metadata:
      labels:
        tier: airflow
        component: webserver
        release: RELEASE-NAME
      annotations:
        checksum/metadata-secret: a3512f27fea8455cdddc51ef650052d74657bcaa16194d24b555417e312d43da
        checksum/pgbouncer-config-secret: da52bd1edfe820f0ddfacdebb20a4cc6407d296ee45bcb500a6407e2261a5ba2
        checksum/airflow-config: 6132d4c762bec566a83667e8a23486fcbc29157811f277b66e6568047f627c14
    spec:
      nodeSelector:
        {}
      affinity:
        {}
      tolerations:
        []
      restartPolicy: Always
      securityContext:
        runAsUser: 50000
        fsGroup: 50000
      initContainers:
        - name: run-airflow-migrations
          image: apache/airflow
          imagePullPolicy: IfNotPresent
          args: ["bash", "-c", "airflow upgradedb || airflow db upgrade"]
          env:    
          # Getting values from the secrets     
            - name: AIRFLOW__CORE__FERNET_KEY
              valueFrom:
                secretKeyRef:
                  name: fernet-key
                  key: fernet-key
            - name: AIRFLOW__CORE__SQL_ALCHEMY_CONN
              valueFrom:
                secretKeyRef:
                  name: airflow-metadata
                  key: connection
            - name: AIRFLOW_CONN_AIRFLOW_DB
              valueFrom:
                secretKeyRef:
                  name: airflow-metadata
                  key: connection
      containers:
        - name: webserver
          image: apache/airflow
          imagePullPolicy: IfNotPresent
          args: ["webserver"]
          resources:
            {}
          volumeMounts:
          # Mounting config, logs, and dags directories to scheduler pods
            - name: config
              mountPath: "/opt/airflow/airflow.cfg"
              subPath: airflow.cfg
              readOnly: true
            - name: logs-pv
              mountPath: "/opt/airflow/logs"
            - name: dags-pv
              mountPath: "/opt/airflow/dags"
          ports:
            - name: webserver-ui
              containerPort: 8080
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            timeoutSeconds: 30
            failureThreshold: 20
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            timeoutSeconds: 30
            failureThreshold: 20
            periodSeconds: 5
          env:
          # Getting values from the secrets           
            - name: AIRFLOW__CORE__FERNET_KEY
              valueFrom:
                secretKeyRef:
                  name: fernet-key
                  key: fernet-key
            - name: AIRFLOW__CORE__SQL_ALCHEMY_CONN
              valueFrom:
                secretKeyRef:
                  name: airflow-metadata
                  key: connection
            - name: AIRFLOW_CONN_AIRFLOW_DB
              valueFrom:
                secretKeyRef:
                  name: airflow-metadata
                  key: connection
      volumes:
      # Specifying volume information for config, logs, and dags
        - name: config
          configMap:
            name: airflow-config
        - name: dags-pv
          persistentVolumeClaim:
            claimName: dags-pvc
        - name: logs-pv
          persistentVolumeClaim:
            claimName: logs-pvc
```


Definition of Webserver service in `webserver-service.yaml`:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: webserver
  # Make sure to specify the right namespace name
  namespace: airflow-on-k8s
  labels:
    tier: airflow
    component: webserver
    release: RELEASE-NAME
    chart: "airflow-1.0.0"
    heritage: Helm
  annotations:
spec:
  type: ClusterIP
  selector:
    tier: airflow
    component: webserver
    release: RELEASE-NAME
  ports:
    - name: airflow-ui
      protocol: TCP
      port: 8080
      targetPort: 8080
```

If you'd like the Webserver to have an external IP, replace `ClusterIP` with `LoadBalancer` in the `webserver-service.yaml`, and you will be able to access from the outside of the cluster without proxies or port forwarding.

Create webserver deployment and service:

```bash
kubectl create -f webserver/webserver-deployment.yaml
kubectl create -f webserver/webserver-service.yaml
```

Check the status of the Airflow UI instance and get its TCP port:

```bash
$ kubectl get services -n airflow-on-k8s
```

```bash
NAME        TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
statsd      ClusterIP      10.0.9.199    <none>        9125/UDP,9102/TCP   24h
webserver   ClusterIP      10.0.9.175    <none>        8080:31003/TCP      19h
```

Map the Airflow UI instance to local port (for example 8080):
```bash
$ kubectl port-forward service/webserver 8080:8080 -n airflow-on-k8s
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

### StatsD

Definition of a StatsD deployment in `statsd-deployment.yaml` file:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: statsd
  # Make sure to specify the right namespace name
  namespace: airflow-on-k8s
  labels:
    tier: airflow
    component: statsd
    release: RELEASE-NAME
    chart: "airflow-1.0.0"
    heritage: Helm
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: airflow
      component: statsd
      release: RELEASE-NAME
  template:
    metadata:
      labels:
        tier: airflow
        component: statsd
        release: RELEASE-NAME
    spec:
      nodeSelector:
        {}
      affinity:
        {}
      tolerations:
        []
      restartPolicy: Always
      containers:
        - name: statsd
          image: astronomerinc/ap-statsd-exporter:0.11.0
          imagePullPolicy: IfNotPresent
          args:
            - "-statsd.mapping-config=/etc/statsd-exporter/mappings.yml"
          resources:
            {}
          ports:
            - name: statsd-ingest
              protocol: UDP
              containerPort: 9125
            - name: statsd-scrape
              containerPort: 9102
          livenessProbe:
            httpGet:
              path: /metrics
              port: 9102
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /metrics
              port: 9102
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 5
```

Definition of a StatsD service in `statsd-service.yaml` file:

```yaml
---
kind: Service
apiVersion: v1
metadata:
  name: statsd
  # Make sure to specify the right namespace name
  namespace: airflow-on-k8s
  labels:
    tier: airflow
    component: statsd
    release: RELEASE-NAME
    chart: "airflow-1.0.0"
    heritage: Helm
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9102"
spec:
  type: ClusterIP
  selector:
    tier: airflow
    component: statsd
    release: RELEASE-NAME
  ports:
    - name: statsd-ingest
      protocol: UDP
      port: 9125
      targetPort: 9125
    - name: statsd-scrape
      protocol: TCP
      port: 9102
      targetPort: 9102

```

Check the status of the StatsD instance and get its TCP port:

```bash
$ kubectl get services -n airflow-on-k8s
```

```bash
NAME        TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
statsd      ClusterIP      10.0.9.199    <none>        9125/UDP,9102/TCP   24h
webserver   ClusterIP      10.0.9.175    <none>        8080:31003/TCP      19h
```

Map the StatsD instance to local port (for example 7000):
```bash
$ kubectl port-forward service/statsd 7000:9102 -n airflow-on-k8s
```

Output:

```bash
Forwarding from 127.0.0.1:7000 -> 9102
Forwarding from [::1]:7000 -> 9102
```

Open the `127.0.0.1:7000` page in your browser to see StatsD page:

![statsd open page](images/statsd-1.png)

![statsd metrics page](images/statsd-2.png)


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
__pycache__  dag_processor_manager  dags  first-dag.py  hello_dag.py  hello_world.py  outfile  scheduler```