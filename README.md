# check_mk

# Installation

To install the Check_MK server to a Kubernetes Cluster (Shared via a Load Balancer Service) apply the kubernetes directory via Kubectl after creating the rbac config supplied by Check_MK:

```
kubectl apply -f monitoring/
kubectl apply -f kubernetes/
```

# Accessing Check_MK

Get the cmkadmin password from the Check_MK deployment logs:

```
kubectl logs -f $(kubectl get pods -n check-mk | grep -v "NAME" | awk {'print $1'}) -n check-mk
```

Get the IP address for the Load Balancer:

```
kubectl get svc -n check-mk
```

Make a Note of the IP Address assigned by MetalLB and access the Check_MK installation:

http://[ip-address]/cmk/check_mk

# Setting up a Backup

The default Deployment configuration will mount a directory called "/backups" which You should configure backups to. The 04-crontab.yaml will offsite these backup files to Your backup server

First select Your SSH Key Pair that You will use for the rsync connection to Your backup server. Make a note of the location for the command below. 

If You wish to create a new keypair for this, use the following (Use the ./backup/ssh directory as this is excluded from git):

```
ssh-keygen -t rsa
## Use Directory /git/check_mk/backup/ssh/id_rsa
```

Now create a Kubernetes secret in the check-mk namespace.

```
kubectl create secret generic backup-ssh-key --from-file=ssh-privatekey=./ssh/id_rsa --from-file=ssh-publickey=./ssh/id_rsa.pub -n check-mk
```

Confirm that the secret has been created in the Namespace:

```
kubectl get secrets -n check-mk
```

You can now schedule the cronjob by applying the Yaml 
```
kubectl apply -f kubernetes/04-cronjob.yaml
```

## Monitoring a Kubernetes Cluster:

Follow these instructions for adding a Kubernetes Cluster to Check_MK Monitoring.

### Get Certificate and Token from Kubectl

Make a note of the following outputs in a Notepad.

Getting the certificate:
```
kubectl get secrets $(kubectl -n check-mk get secret | grep "check-mk" | awk {'print $1'}) -n check-mk -o yaml | grep "ca.crt" | cut -f4 -d' ' | base64 --decode
```

Getting the Token:

```
kubectl -n check-mk describe secrets $(kubectl -n check-mk get secret | grep "check-mk" | awk {'print $1'}) | grep "token:" | awk {'print $2'}
```

### Resolving Permissions Issues on the rbac from Check_MK:
```
kubectl delete clusterrolebinding check-mk
kubectl create clusterrolebinding check-mk --clusterrole=cluster-admin --serviceaccount=check-mk:check-mk
```

### Setting up the Cluster in Check_MK

#### Add the CA Cert

Firstly, add the CA Certificate via WATO:

Global Settings > Site Management > Trusted certificate authorities for SSL > Add a New CA Certificate

Paste the CA cert noted from kubectl.

#### Store the Token as a Password

Add the Token for the Kubernetes user to Password Store via WATO:

Passwords > New Password

Give the Password an appropriate name and paste in the long Token from above.

#### Apply Config 

From the button at the top of the screen, apply the current config to Your Check_MK install at this stage.


#### Configure the Datasource Agent Rule

Now configure the Datasource program in WATO

Host & Service Parameters > Datasource Agents > Kubernetes > Create Rule

Under Kubernetes, select Your Token from the Password store

Select all Kubernetes objects You would like to Monitor (Nodes, Services, Deployments etc)

Select Port 6443 (Unless this is different in Your environment)

Select "Explicit Hosts" and enter the Hostname You will give Your agent (Eg k8s-master.domain.local)

Apply the Config again

#### Add the Master Host

Go to Hosts > Add new Host > Enter the DNS name for Your host

Under "Data Sources", select all configured Special Agents (If You do not have the check_mk agent installed on Your hosts as well)

Select "Save & Go to Services to do a full inventory of Your Cluster"

#### Configuring Node Monitoring

First we must setup Aliases for the Nodes so they are resolvable over the LAN.

Go to Host & Service Parameters > Access to Agents > Hostname translation for piggybacked hosts 

Create a new Rule as follows:

Description: [Node name from kubectl get nodes]

Hostname Translation: Explicity hostname mapping:
    Original Hostname: [Hostname from kubectl get nodes]
    Translated Hostname: FQDN

You should now be able to add Your Nodes as Hosts in Check_MK with Default Monitoring Enabled.

#### Monitoring Deployments

Once Your Kubernetes monitoring has been configured, if You use kubectl to exec to Your container, You should now see piggyback data for Your deployments and services under the following directory:

```
/opt/omd/sites/cmk/tmp/check_mk/piggyback
```

You can now add a Deployment to monitoring as follows:

Firstly, create the following new Rules:

Description: Deployments
Add Label: cmk/kubernetes_object:deployment
Explicit Hosts: ~deployment_

Description: Nodes
Add Label: cmk/kubernetes_object:node
Explicit Hosts: ~node_

Description: Daemon Sets
Add Label: cmk/kubernetes_object:daemon_set
Explicit Hosts: ~daemon_set

More information here: https://checkmk.com/cms_monitoring_kubernetes.html

Now You can create a new host:

Hostname: deployment_deploymentName
IP Address Family: No IP
Check_MK Agent: No agent
Piggyback: Always use and expect piggyback data

Inventory the host and the deployment will show under monitoring.

#### Automatic Deployment monitoring

WIP

https://checkmk.com/cms_web_api_references.html

