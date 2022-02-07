# Medik8s
- [Background](#background)
- [Install Medik8s](#install-medik8s)
- [Install CockroachDB](#install-standalone-cockroachdb)
- [Demo](#demo)
- [uninstall](#uninstall)

for z in $(seq 999); do kubectl get pod -o=custom-columns=NODE:.spec.nodeName,NAME:.metadata.name   -n cockroachdb; sleep 3s; done
## Background
This guide assumes that you have created a cluster with worker nodes in 3 different availability zones. We will use the PoisonPill operator to recover workloads in the event of a Node failure. The specific challange we will address is where we have CockroachDB  deployed as a Deployment backed by a PV and PVC with accessModes set to ReadWriteOnce. We will simulate the node failure by creating a net Network ACL in AWS with no Inbound or Outbound rules configured and assigning it to the node that we want to fail.

Using the Installer Provisioned Infrastructure you can use an install-config.yaml similar to the following:
```
apiVersion: v1
baseDomain: --
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    aws:
      type: m5.xlarge
      zones:
      - us-east-2a
      - us-east-2b
      - us-east-2c
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      zones:
      - us-east-2a
      rootVolume:
      type: m5.xlarge
  replicas: 3
metadata:
  name: aws-us-east-2
platform:
  aws:
    region: us-east-2
  userTags: 
    openshiftClusterID: --
pullSecret: --
sshKey: --
```

We will deploy a Cockroach Deployment to a dedicated node that is NOT in the same region as the control plane, based on the `install-config.yaml` above we could place the workloads on `us-east-2b` or `us-east-2c` since `us-east-2a` is where the control plane lies.

## Install Medik8s
Install medik8s through the operator
```
kubectl create ns poison-pill
kubectl create -f install/medik8s-operator.yaml
```

## Install Standalone CockroachDB
For the installation of the standalone instance of CockroachDB we need to find the name of a node that is not in the same AZ as the control plane. We know the control plane lives in `us-east-2a` based on the `install-config.yaml` so lets find which node lives in `us-east-2b`
```
kubectl get nodes -o wide --show-labels | grep us-east-2b
```

output:
```
ip-10-0-164-180.us-east-2.compute.internal   Ready    worker   116m   v1.22.3+4dd1b5a   10.0.164.180   <none>        Red Hat Enterprise Linux CoreOS 49.84.202111231504-0 (Ootpa)   4.18.0-305.28.1.el8_4.x86_64   cri-o://1.22.1-4.rhaos4.9.gite3dfe61.el8   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=m5.xlarge,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=us-east-2,failure-domain.beta.kubernetes.io/zone=us-east-2b,kubernetes.io/arch=amd64,kubernetes.io/hostname=ip-10-0-164-180.us-east-2.compute.internal,kubernetes.io/os=linux,node-role.kubernetes.io/worker=,node.kubernetes.io/instance-type=m5.xlarge,node.openshift.io/os_id=rhcos,topology.ebs.csi.aws.com/zone=us-east-2b,topology.kubernetes.io/region=us-east-2,topology.kubernetes.io/zone=us-east-2b
```

In this case we see we can use the node with name `ip-10-0-164-180.us-east-2.compute.internal` since changing the ACL will not affect the kube-apiserver.

let's create an environmental variable so we dont have to remember the node name.
```
export TARGET_NODE=ip-10-0-164-180.us-east-2.compute.internal
```

Now, lets install standalone cockroachDB through the helm chart.

```
kubectl create ns cockroachdb
helm install crdb -n cockroachdb --set controlPlaneAZ=us-east-2a charts/standalone-cockroachdb
```

Verify that the pod was installed and running on the correct node after it has time to get into the running state.
```
kubectl get pods -n cockroachdb -o wide -n cockroachdb
```

Make sure the node is the same as the `TARGET_NODE`.

## Demo
So far we have installed `Medik8s` and a standalone instance of CockroachDB deployed by a Deployment with a single replica backed by a PV and PVC with accessModes set to ReadWriteOnce. The next step is to add data to the database and simulate failure on the node and see if we can recover after failure 


Insert some data into the database. Find the pod in the cockroachdb namespace, exec into it and inject some test data.
```
k exec -it <crdb-pod>  -n cockroachdb -- cockroach sql --insecure  --execute="CREATE TABLE roaches (name STRING, country STRING); INSERT INTO roaches VALUES ('American Cockroach', 'United States'), ('Brownbanded Cockroach', 'United States')"
```
expected result
```
INSERT 2
```
Read the data from the database
```
k exec -it <crdb-pod>  -n cockroachdb -- cockroach sql --insecure  --execute="SELECT * FROM roaches;"
```
expected result
```
          name          |    country
------------------------+----------------
  American Cockroach    | United States
  Brownbanded Cockroach | United States
(2 rows)


Time: 1ms
```

We know that the database is operational, now we will simulate failure.  We are going to attach a new Network ACL to the instance running in the `us-east-2b` region, but first, lets create a watch command in the pods -n the cockroachdb namespace and another on the nodes themselves.

**terminal 1**
_watch pods_
```
for z in $(seq 999); do kubectl get pods -o wide -n cockroachdb; sleep 2s; done
```
**terminal 2**
_watch nodes_
```
for z in $(seq 999); do k get nodes; sleep 2s; done
```

Before we simulate failure, lets tell the NodeHealthCheck operator that we only want to wait 10 seconds to reboot an unhealthy node, this is quicker for demo purposes.
This simulates a threshold in which we want the poison-pill to take action.
```
kubectl edit poisonpillconfig poison-pill-config -n poison-pill

set safeTimeToAssumeNodeRebootedSeconds: 10
```

**Create a new Network ACL in the same VPC as your cluster**
1. Go into the AWS console to the instances, select any instance of the nodes on your cluster. Once selected, click on the VPC ID in the instance details. Take a note of the VPC ID.
2. Once you have clicked on the VPC ID, you will see a panel on the left hand side that said **Security** in bold and Network ACLs below, click Network ACLs.
3. After you click Network ACLs, you will be directed to a page where you will see a button **Create Network ACL**, click that.
4. Create a network ACL with a unique name, mine was casey-acl, and for the VPC, use the VPC that your clusters are in, and create the ACL.

**Simulate failure on the target node**
1. Go into the AWS console to the instances, select the instance (node) in the us-east-2b region, once selected, in the details section click the Subnet ID
2. Select the subnet ID, and lick Network ACL on the details table, then **Edit Network ACL association**.
3. Select the newly created ACL

, select the subneta and click Network ACL, and Edit Network ACL Association.


## Uninstall
Clean up medik8s and CockroachDB
```
k delete -f install/
k delete -f ds -n poison-pill --all --force --grace-period=0
k delete ns poison-pill --wait
```

# notes
https://docs.openshift.com/container-platform/4.9/nodes/nodes/eco-poison-pill-operator.html