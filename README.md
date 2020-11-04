# Reverse Proxy Setup for https://www.bsigroup.com/en-GB (Primary URL), https://page.bsigroup.com/ (CDN URL)

### Rancher

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
