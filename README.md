# Qorus Kubernetes

A set of files intended to make it easy to run Qorus in native Kubernetes and in k8s-based cloud environments.
Each folder contains the instructions for a specific k8s deployment type:
- [kubernetes](kubernetes): native Kubernetes (https://kubernetes.io/)
- [azure](azure): Microsoft Azure Kubernetes Service (https://azure.microsoft.com/en-us/products/kubernetes-service/)
- [aws-eks](aws-eks): Amazon Elastic Kubernetes Service (https://aws.amazon.com/eks/)
- [openshift](openshift): native OpenShift (https://www.okd.io/)

**NOTE**: All configuration files are set up to use Qorus Integration Engine(R) Enterprise Edition with the following image: `public.ecr.aws/qorus/qorus-ee:latest`
