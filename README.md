# kubernetes-dse-gke
Deploy DataStax Enterprise (DSE) cluster on a Kubernetes cluster (Local & GKE)

This project provides a set of sample Kubernetes yamls to provision DataStax Enterprise in a Kubernetes cluster environment on your local machine or a GKE cluster for experimental only. It uses "default" namespace in Kubernetes and sample cloud provider's storage class definition. You would modify the yamls according to your own deployment requirements such as namespace, storage device type, etc.

#### Prerequisites:
* Tools including wget, kubectl have already been installed on your machine to execute our yamls.
* Kubernetes server's version is 1.8.x or higher. 

#### 1. Create required configmaps for DataStax Enterprise Statefulset and DataStax Enterprise OpsCenter Statefulset
Docker images provided by DataStax include a startup script that swaps DataStax Enterprise (DSE) and OpsCenter configuration files found in the /config volume directory with the configuration file in the default location on the container. In a Kubernetes deployment, it is enabled through Kubernetes ConfigMap volume to inject configuration data into Pods. 
In this repo, we provide a sample set of DSE and OpsCenter configuration files.  Follow this link for supported configuration files for [DSE](https://github.com/datastax/docker-images/blob/master/server/6.0/files/overwritable-conf-files) and this link for [OpsCenter](https://github.com/datastax/docker-images/blob/master/opscenter/6.5/files/overwritable-conf-files).  You could add additional supported configuration files which are not included in this repo. 
```
$ git clone https://github.com/DSPN/kubernetes-dse-gke

$ cd kubernetes-dse-gke

$ kubectl create configmap dse-config --from-file=common/dse/conf-dir/resources/cassandra/conf --from-file=common/dse/conf-dir/resources/dse/conf

$ kubectl create configmap opsc-config --from-file=common/opscenter/conf-dir/agent/conf --from-file=common/opscenter/conf-dir/conf --from-file=common/opscenter/conf-dir/conf/event-plugins

Sample opscenter.key and opscenter.pem are provided in the ssl folder for self-signed OPSC auth access.
$ kubectl create configmap opsc-ssl-config --from-file=common/opscenter/conf-dir/conf/ssl
```

#### 2. Create your own OpsCenter admin's password using K8 secret
You can update the [opsc-secrets.yaml file's admin_password's value](https://github.com/DSPN/kubernetes-dse-gke/blob/master/common/secrets/opsc-secrets.yaml#L7) with your own base64 encoded password. Use this command **$ echo -n '\<your own password\>' | base64** to generate your base64 encoded password.
```
$ kubectl apply -f common/secrets/opsc-secrets.yaml 
```

#### 3. Choose one of the following deployment options.

##### 3.1 Running DSE + OpsCenter locally on a laptop/notebook
*This yamls set uses emptyDir as DataStax Enterprise data store.*
```
$ kubectl apply -f local/dse-suite.yaml
```

##### 3.2 Running DSE + OpsCenter on Google Kubernetes Engine (GKE) 
Follow the sample commands below to spin up a GKE cluster if you do not have one already. There are minimum cluster requirements that MUST be met for the deployment to succeed. Please ensure you have a GKE cluster meeting these minimums before deploying. The requirements are >**5 nodes of instance type n1-standard-4 with at least 60GB of disk size for each DSE node**.
Run the following command to create a similar GKE cluster:
```
$ gcloud container clusters create k8-10-9-3-gke-n1-std-4 --cluster-version=1.10.9-gke.3 --zone us-west1-b --machine-type n1-standard-4  --num-nodes 5
```
Run the following command to update a kubeconfig file with appropriate credentials and endpoint information to point kubectl at the GKE cluster created above:
```
$ gcloud container clusters get-credentials k8-10-9-3-gke-n1-std-4 --zone us-west1-b
```

*Now you are ready to deploy the yamls set for GKE which uses kubernetes.io/gce-pd provisioner along with pd-ssd persistent disk type*
```
$ kubectl apply -f gke/dse-suite.yaml
```

#### 4. Access the DataStax Enterprise OpsCenter managing the newly created DSE cluster

You can run the following command to monitor the status of your deployment.
```
$ kubectl get all
```
Then run the following command to view if the status of **dse-cluster-init-job** has successfully completed.  It generally takes about 10 minutes to spin up a 3-node DSE cluster.
```
$ kubectl get job dse-cluster-init-job
```
Once complete, you can access the DataStax Enterprise OpsCenter web console to view the newly created DSE cluster by pointing your browser at https://<svc/opscenter-ext-lb's EXTERNAL-IP>:8443 with Username: admin and Password: **datastax1!** (if you use the default OpsCenter admin's password K8 secret)

#### 5. Tear down the DSE deployment
```
$ kubectl delete -f <your cloud platform choice>/dse-suite.yaml (the same yaml file you used in step 3 above)
$ kubectl delete pvc -l app=dse (to remove the dynamically provisioned persistent volumes for DSE in step 3.2)
$ kubectl delete pvc -l app=opscenter (to remove the dynamically provisioned persistent volumes for OpsCenter in step 3.2)
```

### 6. (Optional) Delete the GKE cluster
If you have created a GKE cluster in step 3.2 above and you no longer need it, you can run a similar command like the following to remove the GKE cluster:
```
$ gcloud container clusters delete k8-10-9-3-gke-n1-std-4 --zone us-west1-b
```
