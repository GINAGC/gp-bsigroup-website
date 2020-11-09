# Reverse Proxy Setup for https://www.bsigroup.com/en-GB (Primary URL), https://page.bsigroup.com/ (CDN URL)

## Rancher

Rancher is a container management platform built for organizations that deploy containers in production. Rancher makes it easy to run Kubernetes everywhere, meet IT requirements, and empower DevOps teams.

### Installation steps to deploy K8s cluster on EC2 instances using Rancher
Refer https://github.com/k8-proxy/gp-emma-dataport-website/blob/main/README.md for Server capacity requirement, Client machine tooling requirement and repositories needed.

1. **Associate Elastic IP with the instance created:**

The instance created should have an elastic/static IP associated with it, this is required so that you can mask the failure of an instance or software by rapidly remapping the address to another instance in your account.

Refer [this](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html#using-instance-addressing-eips-associating) for associating elastic IP with your running instance.

2. **Deploy rancher server using docker:**

```
docker run -d --restart=unless-stopped \
  -p 8080:80 -p 8443:443 \
  --privileged \
  rancher/rancher:latest
```

Once the docker is running, it takes few minutes to initialize the server. Once the server is started, access the rancher UI on https://<HOST_IP>:8443

3. **Setup AWS cloud credentials:**

  - Under profile, select "Cloud Credentials" and click on "Add Cloud Credentails". Populate the details of region, access key, secret key, credentails name and save it.

4. **Create an ec2 node template:**

  - Under profile, select "Node templates" and click on "Add template". Choose Amazon ec2 type for node template. 

  - Under Account Access, Choose the region where k8s cluster needs to be deployed. Choose the cloud credentails that is created in step 2.

  - Under Zone and Network, choose the Availability zone and the subnet where cluster nodes should be deployed. Then click next.

  - Under Security groups, choose "standard" to automatically create a security group with required rules for cluster nodes. Then click next.

  - Under Instance, choose the instance type, root disk size etc.

  - Give a name for the node template and click on Create.


5. **Create a K8s cluster:**

  - Go to Clusters in rancher UI.

  - Click on Add cluster. Select the type of cluster to create e.g. AWS EC2, Provide a cluster name and Name prefix for nodes.

  - Select the previously created template from step 3 in the dropdown and give the number of nodes required in the count field.

  - Select etcd, control plane and worker to make sure they are installed in at least 1 node.

  - Click on create button to provision the k8s cluster. This will take some time to get provisioned.


6. **Test the cluster deployment:**

Select and open the cluster to be tested. On the right top, click on "Kubeconfig File" and copy the config file data.
Create a local file called `kubeconfig` and paste the copied data.
Set the KUBECONFIG environment variable to point to that file

```
export KUBECONFIG=kubeconfig
``` 

Verify that the setup works, Commands bellow should generate some output

```
kubectl get nodes
kubectl get all --all-namespaces
``` 

## Deploy adaptation service and proxies (Run all these commands from your machine)

### Clone the needed repositories
```
git clone https://github.com/k8-proxy/icap-infrastructure
git clone https://github.com/k8-proxy/k8-reverse-proxy.git
git clone https://github.com/k8-proxy/s-k8-proxy-rebuild.git
```
### Install ICAP server and adaptation service

Switch to icap-infrastructure repo  
```
cd icap-infrastructure/adaptation
```

Create the Kubernetes namespace
```
kubectl create ns icap-adaptation
```

Create container registry secret
```
kubectl create -n icap-adaptation secret docker-registry regcred	\ 
	--docker-server=https://index.docker.io/v1/ 	\
	--docker-username=<username>	\
	--docker-password=<password>	\
	--docker-email=<email address>
```

copy the updated helm template file mvp-icap-service-configmap.yml from gp-emma-dataport-website repo to the templates folder 
```
cp ../../gp-emma-dataport-website/patch/mvp-icap-service-configmap.yml templates/
```

Install the cluster components
```
helm install . --namespace icap-adaptation --generate-name
```

The cluster's services should now be deployed
```
> kubectl get pods -n icap-adaptation
NAME                                 READY   STATUS    RESTARTS   AGE
adaptation-service-64cc49f99-kwfp6   1/1     Running   0          3m22s
mvp-icap-service-b7ddccb9-gf4z6      1/1     Running   0          3m22s
rabbitmq-controller-747n4            1/1     Running   0          3m22s
```
### Setup squid icap client and nginx for reverse proxy

1. Switch to s-k8-proxy-rebuild repo

  ```bash
    cd ../../k8-reverse-proxy/stable-src/
  ```

Build and push the needed images to your dockerhub registry
  ```bash
    docker build nginx -t <docker registry>/reverse-proxy-nginx:0.0.1
    docker push <docker registry>/reverse-proxy-nginx:0.0.1

    docker build squid -t <docker registry>/reverse-proxy-squid:0.0.1
    docker push <docker registry>/reverse-proxy-squid:0.0.1
  ```
2. Setup cert manager

Switch to the gp-bsigroup-website repo
  ```bash
    cd ../../gp-bsigroup-website/stable-src/
  ```
Create cert manager namespace
  ```bash
    $ cd ../../icap-infrastructure
    $ kubectl create namespace cert-manager
  ```

Install cert manager controller
  ```bash
    $ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.12.0/cert-manager.yaml
  ```

Make sure the pods are running
  ```bash
    $ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.12.0/cert-manager.yaml
  ```

Setup letsencrypt issuer
  ```bash
    $ kubectl -n icap-adaptation apply -f manifest.yml
  ```
3. Setup reverse proxy k8s deployment repo

File value.yaml has been modified to include some features :

  ```bash
    Change allow domain to target bsigroup.com, page.bsigroup.com
    Set default ICAP_URL point to Glasswall offcial ICAP server
    Enable Ingress
    Auto deploy cert-manager for domain bsigroup.com.glasswall-icap.com
  ```
Setup the services (If you are proxying another site, use that site url instead of the one defined bellow)
  ```bash
    helm upgrade --namespace icap-adaptation upgrade --install \
	--set image.nginx.repository=<docker registry>/reverse-proxy-nginx \
	--set image.nginx.tag=0.0.1 \
	--set image.squid.repository=<docker registry>/reverse-proxy-squid \
	--set image.squid.tag=0.0.1 \
	--set image.icap.repository=<docker registry>/reverse-proxy-c-icap \
	--set image.icap.tag=0.0.1 \
	reverse-proxy chart/
  ```
Get the ingress external IP
 ```bash
    $ kubectl -n icap-adaptation get ing
  ```  

Record the ingress ip from the last step and add it to your host file
 ```
    <ingress IP address>	 bsigroup.com.glasswall-icap.com www.bsigroup.com.glasswall-icap.com page.bsigroup.com.glasswall-icap.com
