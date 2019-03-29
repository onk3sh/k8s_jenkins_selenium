# Jenkins and Selenium Grid setup on Google Container Registry in GCP via Kubernetes

## Prerequisites
We need to first set the compute zone and then clone the code into the Cloud Shell:
```
gcloud config set compute/zone us-central1-f
```

## Step 0 : Provisioning Jenkins via Helm
The following steps will install and configure jenkins on the cluster.

Download and install the helm binary
```
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.9.1-linux-amd64.tar.gz
```

Unzip the file in the cloud shell
```
tar zxfv helm-v2.9.1-linux-amd64.tar.gz
cp linux-amd64/helm .
```

Add self as cluster administrator so that jenkins permissions can be editied by self.
```
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
```
Grant Tiller, the server side of Helm. the cluster admin role in the cluster.
```
kubectl create serviceaccount tiller --namespace kube-system
kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```

Initialize Helm
```
./helm init --service-account=tiller
./helm update
```

Verify installation by checking the version of helm
```
./helm version
```
## Step 1 : Configure and Install Jenkins
Use Helm cli to deploy jenkins.
```
./helm install -n cd stable/jenkins -f jenkins/values.yaml --version 0.16.6 --wait
```

Verify that the pod is created in the cluster for jenkins
```
kubectl get pods
```

Setup port forwarding to the Jenkins UI from the cloud shell
```
export POD_NAME=$(kubectl get pods -l "component=cd-jenkins-master" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
```

Check that the Jenkins Service was created properly.
```
kubectl get svc
```

### Step 1.1 Connect to Jenkins UI
Jenkins will automatically create an admin password. Execute the following command to retrive the same.
```
printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```

Next, To connect to Jenkins UI, click on the web Preview button in the cloud shell and then click on Preview on port 8080.

## Step 2 : Create the K8s cluster for the grid
Use the below command to create the cluster inside the Google Cloud Platform
```
gcloud container clusters create jenkins-selenium-grid \
--num-nodes 2 \
--machine-type n1-standard-2 \
--scopes "https://www.googleapis.com/auth/projecthosting,cloud-platform"
```
Note: This task is time consuming and will take upto 5 minutes to complete.

## Step 3: Get the deployment files into the GCP ecosystem
Clone the code using the gcp console and open the k8_selenium_grid folder
```
git clone https://github.com/onk3sh/k8s_selenium_grid
cd k8s_selenium_grid
```

## Step 4: Create the deployments and the service for the selenium grid 
Creating the pod for the grid hub using the deployment file copied in the previous step.
```
$ kubectl create -f deployment/selenium-hub-deployment.yaml
```

Lets create a service for nodes to connect to and launching selenium hub as a service
```
$ kubectl create -f service/selenium-hub-svc.yaml
```

Expose the service across the internet using an external IP:
```
kubectl expose deployment selenium-hub --name=selenium-hub-external --labels="app=selenium-hub,external=true" --type=LoadBalancer
```
Wait for a few minutes to execute this check if the above IP has been exposed:
```
export INTERNET_IP=`kubectl get svc --selector="app=selenium-hub,external=true" --output=template --template="{{with index .items 0}}{{with index .status.loadBalancer.ingress 0}}{{.ip}}{{end}}{{end}}"`

curl http://$INTERNET_IP:4444/
```
We can also get the external IP by getting the services
```
kubectl get services
```

## Step 5: Spin up Child Chrome Nodes inside the cluster
Now since hub is up and exposed lets spin up nodes and register into hub.
```
$ kubectl create -f deployment/selenium-node-chrome-deployment.yaml
```
By default 2 pods have been configured inside the deployment file to be launched once the above step is executed.
```
kubectl scale deployment selenium-node-chrome --replicas=10
```
We can use the above step to scale the grid.

Now nodes are up and registered to hub created above can be seen in exposed selenium-hub’s console. 

### End Step: Running tests on the grid
Now consume the grid in any selenium test using the external IP or inside a Jenkins job using the internal IP
Example: External IP consumption
```
http://$INTERNET_IP:4444/wd/hub
```
