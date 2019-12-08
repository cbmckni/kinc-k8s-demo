
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

### Access VM

First, download and install VNC Viewer: [link](https://www.realvnc.com/en/connect/download/viewer/)

Open the application, then enter your assigned IP in the address bar at the top. 

Enter your assigned **VNC** password when prompted.

You should now have access to your own personal VM!

Go to https://github.com/cbmckni/techex-demo and clone the repo to your VM's desktop:

`cd ~/Desktop && git clone https://github.com/cbmckni/techex-demo.git`

## 1. Deploy Kubernertes Cluster Using Cisco Container Platform

There are two ways to use CCP: the web GUI and command line interface.

For this workshop, we will use the CLI. Instructions for using the web interface are below. 

### Create a K8s Cluster Using the CCP Command Line Interface

There should be an open terminal on the VM that is already running the CCP CLI environment.

**If there is not a CCP CLI terminal open, or you closed the terminal, run the following in a new terminal:**

`cd Desktop/ccp_cli`

`source venv/bin/activate`

This will activate the CCP CLI environment.



We will create a Kubernetes cluster with 1 master node and 3 workers. Give the cluster the same name as your CCP username, or *workshopX*

To create the cluster, run:

`ccp create cluster -name <CCP_Username> -master 1 -m-mem 32768 -worker 3 -w-mem 32768 -vcpu 4 -load 1

The command should run until your cluster has been created.

Next, run `ccp cluster list` to list the clusters, find your cluster and copy the UUID.

To download the kubeconfig for your cluster, run:

`ccp create kubeconfig -uuid <UUID> -file kubeconfig`

Finally, move the kubeconfig to your Downloads folder: 

`mv kubeconfig.yaml ~/Downloads`

### [ALTERNATIVE] Create a K8s Cluster Using the CCP Web Interface

Open Firefox and click the Cisco Container Platform link at the top of the browser.

Login with your **CCP** username and password.

Click *New Cluster*.

Input the following for the dialog boxes:
 - Infrastructure provider: vsphere
 - Cluster name: same as username you logged in with

Click *Next* at the bottom
 - Data-center: csn-hx
 - Cluster: cns-hx-c220
 - Network: cpn-colab
 - Datastore: hx-datastore 2
 - Template: ccp-tenant-image-1.14.6-ubuntu18-5.0.0

Leave everything else the same and click next at the bottom
 - master: increase the vcpu’s the 4 and the memory to 32gb
 - worker: increase the nodes to 3, vcpu’s the 4, and the memory to 32gb
 - vm username: same as username you logged in with
 - load balancer ip’s- increase to 2
 - subnet: cpn-sjc-colab-subnet
Leave everything else the same and click next twice

Look over the summary of settings and click finish to initialize creating the cluster(this
will take a few minutes)

Once cluster is created, click the arrow under the actions column for your respective
cluster and click download kubeconfig

You now have access to your own personal K8s cluster!

## 2. Create Persistant Data Storage to Host Workflow Data

Now that you have a K8s cluster, it is time to access it from the VM.

First, tell the VM where to look for the Kubernetes configuration file:

`echo 'export KUBECONFIG=~/Downloads/kubeconfig.yaml' >> ~/.bashrc && source ~/.bashrc`

Test your connection with:

`kubectl config current-context`

The output should match the name of your cluster.

Now it is time to provision a NFS server to store workflow data. We will streamline this process by using Helm.

Update Helm's repositories(similar to `apt-get update)`:

`helm repo update`

Next, install a NFS provisioner onto the K8s cluster to permit dynamic provisoning for 10Gb of persistent data:

`helm install kf stable/nfs-server-provisioner \`

`--set=persistence.enabled=true,persistence.storageClass=standard,persistence.size=10Gi`

Check that the `nfs` storage class exists:

`kubectl get sc`

Next, deploy a 8Gb Persistant Volume Claim(PVC) to the cluster:

`cd ~/Downloads/techex-demo`

`kubectl apply -f task-pv-claim.yaml`

Check that the PVC was deployed successfully:

`kubectl get pvc`

Finally, login to the PVC to get a shell, enabling you to view and manage files:

`nextflow kuberun login -v task-pv-claim`

**Open a new terminal window.**

## 3. Get Access to Grafana Visualization Interface

Get username and password to access Grafana via the following commands using kubectl.

Get username: 

`export GRAFANA_USER=$(kubectl get secret ccp-monitor-grafana -n ccp -o=jsonpath='{.data.admin-user}' \`

`| base64 --decode)`

Get password: 

`export GRAFANA_PASSWORD=$(kubectl get secret ccp-monitor-grafana -n ccp -o=jsonpath='{.data.admin-password}' \`

`| base64 --decode)`

Print username: 
`echo $GRAFANA_USER`

Print password:
`echo $GRAFANA_PASSWORD`
 
 - Go to firefox and click on the CCP link at the top of the browser, login with access credentials given.
 - At the top of the GUI where it says version 3, click the box and select version 2.
 - For your respective cluster, go to the actions column, click on the appearing arrow, and click Grafana.
 - Copy and paste the username and password from the echo commands above.

## 4. Deploy a Genomic Workflow to Your Cloud

**In a new terminal window....**

Go to the **techex-demo** folder:

`cd ~/Downloads/techex-demo`

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

### 5. Retreive and Visualize Gene Co-expression Network

Copy the output of KINC from the PVC to your VM:

`./kube-save.sh task-pv-claim output`

Open Cytoscape. (Applications -> Other -> Cytoscape)

Go to your desktop and open a file browsing window, navigate to the output folder:

`cd ~/Desktop/techex-demo/output/Yeast`

Finally, drag the file `Yeast.coexnet.txt` from the file browser to Cytoscape!

The network should now be visualized! 






