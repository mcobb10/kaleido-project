# kaleido-project

## Setup

### Prerequisites
* minikube is installed on your machine
* Increase minikube resources to cpus: 4, memory: 4GB, disk-space: 64GB (This is probably overkill but I had resources to spare and ran into vm lockups before this increase.)
```
minikube config set cpus 4
minikube config set memory 4096
minikube config set disk-size 64GB
```
* Racecourse webapp is built into a nodejs image named test/racecourse (this can be changed in racecourse_deployment.yaml if needed)
* Quorum-Tools images have been built and quorum-tools/examples/setup.sh has been ran

### Launching Cluster
* Run `minikube start`
* Run `minikube addons enable ingress`
* Copy test/racecourse, jpmorganchase/quorum, and jpmorganchase/constellation from local machine to minikube vm: 
```
sudo docker save "image-name" | pv | (eval $(minikube docker-env) && docker load)
```
> Note: This command is basically saving the local docker image to a file then piping it into the minikube docker instance and loading it. The process may take several minutes to complete based on the size of the image. It would be more ideal here to use a docker registry but this worked for testing purposes.
* Run 'kubectl apply -f filename.yaml' for all yaml configuration files in the kaliedo-project directory
* When the bootnode and node stateful sets try to start they will error out but will create their persistent volumes
* Mount the quorum-tools/examples/tmp directory in minikube and copy needed files over (replace disk_num with number 0-5):
```
(terminal 1) minikube mount ~/quorum-tools/examples/tmp:/tmp/hostpath_pv
(terminal 2) minikube ssh
(terminal 2) sudo cp -r /tmp/hostpath_pv/qdata_<disk_num>/* /data/qdata_<disk_num>/ 
```
* Once the needed files are in place everything should eventually restart correctly

## Discussion of Architecture

This setup can be thought of as 3 services: racecourse-service, bootnode-service, and node-service. 

The racecourse-service is backed the racecourse-deployment that runs a single pod racecourse webapp and is accessed by the racecourse-ingress that directs web browser traffic hitting the minikube vm through to the racecourse pod on port 3000.

The bootnode-service is backed by a stateful set (bootnode-set) with replicas set to 1. The pod created by this set is running a single quorum container. This set utilizes a corresponding PVC and PV that provide it a persistent volume. It has a clusterIP of 10.96.0.100 to be used by the quorum nodes for establishing the p2p network. 

The node-service is backed by 5 stateful sets (node-set-1 to 5) with replicas set to 1. The pod that is created by these sets consists of a quorum container and a constellation container. These sets also utilize corresponding PVC and PV definitions that provide the various sets a persistent volume. The service has a clusterIP of 10.96.0.99 to allow it to be contacted by the racecourse webapp easily.

## Discussion of Design Decisions

The racecourse webapp is running as a deployment since it is generally stateless and only connects back to the node-service for running transactions against the blockchain. The bootnode and quorum nodes need to have persistent volumes to maintain various files for operation (i.e. nodekeys, blockchain, ect) so I have created them as stateful sets. Normally the stateful sets would contain a volumeClaimTemplate but I ran into issues with minikube and mounting the local directories that back the persistent volumes so I eventually factored out the PVCs and PVs to make sure I was specifying everything correctly. This is also why I have opted for 5 stateful sets for the nodes instead of one that has 5 replicas. To make sure I had all storage issues ironed out, I split the needed pods out into their own stateful set and assigned them all a particular PV instead of allowing them to provision dynamically.

I am confident the majority of the issues I ran into (storage, command hangs, VM lockup) were due to my unfamiliarity with minikube and having to get reacquainted with Kubernetes after not using it in my daily role at Bronto. This definitely was a great project to brush up on my skills and please feel free to ask any questions you have about the setup.
