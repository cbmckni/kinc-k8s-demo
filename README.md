
Internet2 Tech Exchange 2019: Scaling Genomics Workflows with Kubernetes Hybrid Cloud Solutions
====

Clemson University, Google and Cisco are collaborating to accelerate genomics research with Kubernetes based cloud solutions. The systems genetics lab at Clemson, led by Dr. Alex Feltus (professor in the Clemson Dept. of Genetics & Biochemistry, faculty in the Clemson Center for Human Genetics and Biomedical Data Science & Informatics program, and Internet2 Board of Trustees member) is engaging CU  students with the Google and Cisco teams to deploy a scalable Kubernetes solution to secure and process gigantic DNA data sets in a hybrid cloud environment. A core Feltus lab goal is to identify petascale research computing models that facilitate “normal” research labs (or collaborative group) in the unfolding transition of life science data analysis from the Excel-scale to the exascale. One model consists of mixing commercial cloud (e.g. GCP, CHC) with public compute resources (e.g. distributed research platform like The National Research Platform).

The Demo:

We will cover the complete deployment life cycle that enables scientific workflows to be run in the cloud. This includes:
 - Deployment of a Kubernetes(K8s) cluster using Cisco Container Platform(CCP). 
 - Creating a persistent NFS server to store workflow data, then loading a Gene Expression Matrix(GEM) onto it.
 - Deploying Knowledge Independent Network Construction(KINC), a genomic workflow, to the K8s cloud.
 - Downloading the resulting Gene Coexpression Network(GCN) from the NFS server, then visualizing the network.

## 0. Prerequisites

The following software is necessary to participate in the demo:
 - Cisco Container Platform 
 - Helm - Kubernetes Package Management Utility
 - kubectl - Kubernetes CLI 
 - Nextflow - Workflow Manager
 - Java
 - Cytoscape - Network Visualization Tool
 - Files/scripts from this repo.

To streamline the demo, all software has been packaged into a virtual machine that has been replicated for each user. 

An additional requirement is access to the CCP environment where the K8s clusters will be created.

**If you do not have your CCP login credentials and access to your personal VM, please let us know.**

## 1. Deploy Kubernertes Cluster Using Cisco Container Platform


## 2. Create Persistant Data Storage to Host Workflow Data

Now that you have a K8s cluster, it is time to access it from the VM.

First, tell the VM where to look for the Kubernetes configurtion file:

`export KUBECONFIG=~/Downloads/kubeconfig.yaml`

Test your connection with:

`kubectl config current-context`

The output should match the name of your cluster.

Now it is time to provision a NFS server to store workflow data. We will streamline this process by using Helm.

Update Helm's repositories(similar to `apt-get update)`:

`helm repo update`

Next, install a NFS provisioner onto the K8s cluster to permit dynamic provisoning for 10Gb of persistent data:

`helm install kf stable/nfs-server-provisioner \
--set=persistence.enabled=true,persistence.storageClass=standard,persistence.size=10Gi`

Check that the `nfs` storage class exists:

`kubectl get sc`

Next, deploy a 8Gb Persistant Volume Claim(PVC) to the cluster:

`kubectl apply -f task-pv-claim.yaml`

Check that the PVC was deployed successfully:

`kubectl get pvc`

Finally, login to the PVC to get a shell, enabling you to view and manage files:

`nextflow kuberun login -v task-pv-claim`

**Open a new terminal window.**

## 3. Deploy Genomic Workflow to Your Cloud

**In a new terminal window....**

Go to the `techex-demo` folder:

`cd ~/Desktop/techex-demo`

Give Nextflow the necessary permissions to deploy jobs to your K8s cluster:

```
kubectl create rolebinding default-edit --clusterrole=edit --serviceaccount=default:default 
kubectl create rolebinding default-view --clusterrole=view --serviceaccount=default:default
```

Load the input data onto the PVC:

`./kube-load.sh task-pv-claim input`

Deploy KINC using `nextflow-kuberun`:

`nextflow kuberun systemsgenetics/kinc-nf -v task-pv-claim`

**The workflow should take about 10-15 minutes to execute.**

#### 4. Retreive and Visualize Gene Co-expression Network

Copy the output of KINC from the PVC to your VM:

`./kube-save.sh task-pv-claim output`

Open Cytoscape. (Applications -> Other -> Cytoscape)

Go to your desktop and open a file browsing window, navigate to `~/Desktop/techex-demo/output/Yeast`.

Finally, drag the file `Yeast.net` from the file browser to Cytoscape!

The network should now be visualized! 






