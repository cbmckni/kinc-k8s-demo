
Internet2 Tech Exchange 2019: Scaling Genomics Workflows with Kubernetes Hybrid Cloud Solutions

#### 0. Deploy Kubernertes Cluster Using Cisco Container Platform


#### 1. Create Persistant Data Storage to Host Workflow Data

export KUBECONFIG=~/Downloads/kubeconfig.yaml

helm repo update

helm install kf stable/nfs-server-provisioner \
--set=persistence.enabled=true,persistence.storageClass=standard,persistence.size=10Gi

kubectl get sc

kubectl apply -f task-pv-claim.yaml

kubectl get pvc

nextflow kuberun login -v task-pv-claim

#### 2. Deploy Genomic Workflow to Your Cloud

kubectl create rolebinding default-edit --clusterrole=edit --serviceaccount=default:default 
kubectl create rolebinding default-view --clusterrole=view --serviceaccount=default:default

./kube-load.sh task-pv-claim input

nextflow kuberun systemsgenetics/kinc-nf -v task-pv-claim

#### 3. Retreive and Visualize Gene Co-expression Network

./kube-save.sh task-pv-claim output







