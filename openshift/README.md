# Qorus Openshift

A set of files intended to make it easy to run Qorus in Openshift.

## Minishift

The easiest way to setup a Openshift cluster is to use Minishift. This will create a small 1-node cluster in a VM, which is good enough for testing.

First you have to install Minishift via a package manager or some other method viable for your OS. Then run the following in shell and wait:
```
minishift start
```

After everything is loaded, you can connect to the VM by using this (for debugging purposes):
```
minishift ssh
```

## HostPath

To run Postgresql on the VM with persistent storage, you will need to set some permissions.

On the VM:
```
minishift ssh
mkdir /mnt/data # or whatever hostPath you've chosen in the qorus-postgre-storage.yaml
sudo chcon -Rt svirt_sandbox_file_t /mnt/data
```

On your local machine:
```
oc adm policy add-scc-to-user anyuid -n myproject -z default
```

## NFS

To run Qorus in Openshift you need to setup NFS on the VM if you are using Minishift. Make sure that NFS is installed on the VM.
You will also need to set up the following permission:
```
oc adm policy add-scc-to-user hostmount-anyuid -n myproject -z default
```

### Exported Directories

1. Create a directory somewhere in your home which will be used for the shared directories. Also create `etc`, `log`, `init` and `user` subdirectories in it.

2. Edit the file `/etc/exports` as root to setup the shared directories for NFS.

#### Linux /etc/exports example:
```
/path/to/omqdir/user  192.168.99.102(rw,sync,no_subtree_check,no_root_squash)
/path/to/omqdir/etc   192.168.99.102(rw,sync,no_subtree_check,no_root_squash)
/path/to/omqdir/log   192.168.99.102(rw,sync,no_subtree_check,no_root_squash)
/path/to/omqdir/init  192.168.99.102(rw,sync,no_subtree_check,no_root_squash)
```

#### Mac /etc/exports example:
```
/path/to/omqdir -alldirs -maproot=root
```

### (Re)starting NFS

NFS can be (re)started using `systemctl` as root:
```
systemctl restart nfs-server
```

If you want NFS to run automatically after every restart of your computer, use the following:
```
systemctl enable nfs-server
```

If NFS is already running and you modified the NFS configuration, you can simply reload the config with the following (as root):
```
exportfs -arv
```

## Openshift Setup

Make sure that your Openshift cluster is running and `oc` is correctly set up for it. If you use Minishift, it will be taken care of by it.

### Database Preparation

Qorus requires a DB to run; to set up the DB, follow the instructions in one of the following two sections to set up
Qorus's system DB for use with Openshift.

#### DB: PostgreSQL Single-Node in Openshift

This example will set up a single-node PostgreSQL 12 instance running in a Openshift pod to be used as the Qorus
system DB.  This way, the DB is available while the pod is running.  If its storage is deleted, the data in the DB is
lost, and Qorus will need to be reconfigured for a new DB before launching Qorus again.

To run the example PostgreSQL single-node configuration, simply apply its storage (persistent volume for DB files),
deployment, and service yaml files as follows (you will need to be logged in as system:admin to create a permanent volume):
```
oc login -u system:admin
oc apply -f qorus-postgres-storage.yaml
oc apply -f qorus-postgres-service.yaml
oc apply -f qorus-postgres-deployment.yaml
```

You can check that PostgreSQL is running using the following:
```
oc get pods
```

<dl>
  <dt><b>&#9995 Note:</b></dt>
  <dd>A single node DB does not provide fault-tolerance and failover capabilities needed for mission-critical
      integration solutions.  Use a multi-node PostgreSQL setup in Openshift or an external DB cluster in
      mission-critical deployments</dd>
</dl>

#### DB: External DB

Create an `options` file in your NFS-mounted `etc` directory (ex: `/path/to/omqdir/etc/options`) with the following settings:

```
qorus.systemdb: <DB-URI>
qorus.instance-key: <INSTANCE NAME>
qorus-client.omq-data-tablespace: <DATA TABLESPACE NAME>
qorus-client.omq-index-tablespace: <INDEX TABLESPACE NAME>
qorus.auto-recover: true
qorus.http-secure-server: 8011
qorus.http-secure-certificate: $OMQ_DIR/etc/cert.pem
qorus.http-secure-private-key: $OMQ_DIR/etc/key.pem
```

Replace the values in angle brackets with the values for your Qorus instance in Openshift; see the [System Options documentation](https://qoretechnologies.com/manual/qorus/current/qorus/systemoptions.html) for more information about Qorus system options.

The most critical value above is the `qorus.systemdb` option, which describes to Qorus how to reach the system DB.
All Qorus Openshift images need to have access to the given database.

Example database configuration value: `qorus.systemdb: pgsql:omq/omq@omq%hostname:5432`

<dl>
  <dt><b>&#9995 Note:</b></dt>
  <dd>Oracle and PostgreSQL databases require tablespace definitions; MySQL/MariaDB do not.  Remove the tablespace
      options for MySQL/MariaDB schemas.  If you did not create tablespaces for a PostgreSQL database, you can use
      `pg_default` for the tablespace name.</dd>
</dl>

### Qorus Service

Qorus needs a Openshift service in order to be reachable outside of the cluster. The service is configured in the `qorus-service.yaml` file. You can edit port settings in the file if necessary.

After you're done with the configuration, you can start the service by applying it:
```
oc apply -f qorus-service.yaml
```

### Qorus

Qorus uses statefulset for Qorus masters and a deployment with a single pod for `qorus-core`. These are described in 2 yaml files:
* `qorus-statefulset.yaml`
* `qorus-core-deployment.yaml`

These yaml files have to be edited before using them as described in the following sections.

#### NFS Volume Configuration

NFS server IP address and paths to the pre-created NFS directories need to be edited to match your setup in both `qorus-statefulset.yaml` and `qorus-core-deployment.yaml`. The IP address should be the address of your computer, on which Minishift is running.

<dl>
  <dt><b>&#9995 Note:</b></dt>
  <dd>The <b><tt>volumes</tt></b> section in both files needs to be identical.</dd>
</dl>

#### Setup Authentication to a Private Repository (optional)

If the container images in each file are pulled from a private repository, then you have to create a credential secret used to pull the images as follows:
```
oc create secret docker-registry qorus-secret --docker-server=https://registry.qoretechnologies.com/v1/ --docker-username=username --docker-password=password --docker-email=email@example.com
```

Then add a reference to the secret to the pod definition. For example in `qorus-statefulset.yaml`:
```
  ...
    - name: qorus-init
      nfs:
        server: 192.168.0.28
        path: "/Users/david/src/Qorus/openshift/init"
    imagePullSecrets:
    - name: qorus-secret
```

<dl>
  <dt><b>&#9995 Note:</b></dt>
  <dd>The <b><tt>imagePullSecrets</tt></b> section in both files needs to be identical</dd>
</dl>

## Starting Qorus

After you've configured the `qorus` statefulset and the `qorus-core` deployment, you can start Qorus using the following steps:

1. Load Qorus statefulset:
```
oc apply -f qorus-statefulset.yaml
```

2. Load qorus-core deployment:
```
oc apply -f qorus-core-deployment.yaml
```

## Scaling Qorus

Once the statefulset is running, you can scale it:
```
oc scale statefulset/qorus --replicas=3
```

## Auto-scaling Qorus

For auto-scaling the Qorus statefulset, we can use the Openshift built-in Horizontal Pod Autoscaler.
The file `qorus-hpa.yaml` contains it's configuration. This is not enough by itself though.

We also need a metrics-server which will store metrics of running pods and provide that info to the autoscaler (HPA),
which will then use it to decide if any scaling is necessary.

The autoscaler requires limits and requests for resources (CPU and memory) to be set for the Qorus statefulset.
These limits and requests are set in the `qorus-statefulset.yaml` file and can be modified to fit the user's needs.

### metrics-server

To load up the metrics server, simply apply it's configuration file:
```
kb apply -f metrics-server.yaml
```

This configuration file is a modified version of the official configuration from the [metrics-server ](https://github.com/openshift-sigs/metrics-server) Github page:
https://github.com/openshift-sigs/metrics-server/releases/download/v0.3.6/components.yaml

The configuration is mostly the same, except 2 arguments have been added for the metrics-server deployment:
* `--kubelet-insecure-tls`
* `--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname`

These arguments are necessary for Minishift deployment. For production deployments, please consult the metrics-server and Openshift documentation to ensure a safe setup.

### Autoscaler (HPA)

To load up the autoscaler, apply it's configuration file:
```
kb apply -f qorus-hpa.yaml
```

If you want to change it's configuration, modify the YAML configuration file and simply re-apply it.

To see basic info about the autoscaler, you can use the following command:
```
oc get hpa qorus
```

## Checking Errors

You can check for errors anytime with one of the following (depending on what is failing):
```
oc describe statefulset qorus
oc describe pod qorus-0
oc describe pod qorus-core-xxxx
oc describe hpa qorus
```

You can also try to check pod logs:
```
oc logs qorus-0
oc logs qorus-core-xxxx
```

## Qorus Web UI

After everything is running, Qorus UI can be accessed through web browser using the cluster/Minishift IP and the HTTPS port exposed by the Qorus service (`31011` by default), for example: https://192.168.99.102:31011

If you're using Minishift, you can find out the cluster IP using the following command:
```
minishift ip
```

## Change Qorus Options / Install New HTTPS Certificate / Restart qorus-core

1. Edit the `etc/options` file.

2. Restart qorus-core pod:
```
oc replace --force -f qorus-core-deployment.yaml
```

## Shutting Down

Delete the `qorus-core` deployment first (for an orderly shutdown of the Qorus application) and then the `qorus` statefulset, and wait until all Qorus pods are terminated (services can be reused):
```
oc delete deployment/qorus-core --wait=true
oc delete statefulset/qorus
```

Alternatively in one line:
```
oc delete deployment/qorus-core --wait=true && oc delete statefulset/qorus
```

## Reset

If there is any problem or you just want a fresh setup, do the following:

1. Delete the `qorus-core` deployment first (for an orderly shutdown of the Qorus application) and then the `qorus` statefulset, and wait until all Qorus pods are terminated (services can be reused):
```
oc delete deployment/qorus-core --wait=true
oc delete statefulset/qorus --wait=true
oc delete deployment/qorus-postgres
```

Alternatively in one line:
```
oc delete deployment/qorus-core --wait=true && oc delete statefulset/qorus --wait=true && oc delete deployment/qorus-postgres
```

2. Clean the `etc`, `log`, `init` and `user` shared directories.

3. Start from the beginning of [Openshift Setup](#openshift-setup).
