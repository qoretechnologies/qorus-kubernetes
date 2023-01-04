# Qorus on Amazon EKS

Files in this subdirectory and this info extend those in the kubernetes subdirectory.

We need to install the files

```
kubectl apply -f qorus-postgres-storage.yaml
kubectl apply -f qorus-postgres-service.yaml
kubectl apply -f qorus-postgres-deployment.yaml
kubectl apply -f qorus-service.yaml
kubectl apply -f qorus-roles.yaml
kubectl apply -f metrics-server.yaml
```

and create the secret to be able to access the qorus images

```
kubectl create secret generic qorus-secret --from-file=.dockerconfigjson=$HOME/.docker/config.json --type=kubernetes.io/dockerconfigjson
```

and create PersistentVolumes + PersistentVolumeClaims for mounting the NFS directories etc, init, log, user

```
kubectl apply -f qorus-pv-claim-etc.yaml
kubectl apply -f qorus-pv-claim-init.yaml
kubectl apply -f qorus-pv-claim-log.yaml
kubectl apply -f qorus-pv-claim-user.yaml
```

the qorus files differ from those in the kubernetes subdirectory only in the urls:

```
kubectl apply -f qorus-core-deployment.yaml
kubectl apply -f qorus-statefulset.yaml
```

The instance can be accessed by

```
ssh -i <.pem-file> ec2-user@ec2-18-185-56-110.eu-central-1.compute.amazonaws.com
```

where the pem-file contains the private part of a key pair - one of those listed in

```
https://eu-central-1.console.aws.amazon.com/ec2/v2/home?region=eu-central-1#KeyPairs
```

The containers can be entered by

```
kubectl exec -it qorus-core-xxxx -- bash
```

or

```
kubectl exec -it qorus-0 -- bash
```
