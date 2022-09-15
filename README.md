# Deploy new Kubernetes Cluster

We have deployed a new `hawk-2` cluster to the `sth` data center. This document contains the main steps we have followed. These steps will be applicable for deploying new clusters or migrating existing clusters between the data centers (e.g `thn` to i`sth` and vice versa). 

## 1. Cluster Setup

Following are the main repositories involved in the cluster setup.

1. [capi-clusters](https://github.com/pagero/capi-clusters)
2. [helm-charts](https://github.com/pagero/helm-charts)
3. [deployments](https://github.com/pagero/deployments)
4. [ops](https://github.com/pagero/ops)

The relationship between these repositories is defined in the following figure. Please refere this diagram when reading thourh the below sections.

![Alt text](./figures/capi-cluster-repos.png?raw=true "capi cluster repos")

### 1.1. capi-clusters

We have used `Cluster-API Provider Openstack(CAPO)` to deploy the cluster in the Nordlo k8s environment. The capi-clusters define the manifest of clusters with Cluster-API.  these cluster configurations are linked to the deployments repo on GitHub. Following are the cluster configurations and Cluster-API configurations related to the `hawk-2` cluster. The best way to add and create these config files is to copy them from another cluster and replace the cluster names and other configurations. For example, in our case, we have copied `hawk-1` cluster configs(in `hawk-1` folder) which deploy in `thn` and renamed the fields to hawk-2 and modified the configuration as needed(for example adding `sth` related configs)

1. `audit-policy.yaml`
2. `cloud-config-secret.yaml`(changed the fields according to the sth). These secrets will be automatically generated when deploying the cluster. no need to do manual creation of secrets
3. `cluster-bootstrap-crs-secret.yaml`
4. `cluster-bootstrap-crs.yaml`
5. `cluster.yaml`(Defines cluster-api management cluster manifest, needs to change the `cirdBlocks` according to the sth configuration. sth related `cirdBlock` configuration can be found in another cluster which deploys in the sth, e.g prod-2) 
```
clusterNetwork:
  services:
    cidrBlocks:
      - 10.227.64.0/18
  pods:
    cidrBlocks:
      - 192.168.0.0/16
  serviceDomain: cluster.local
```
6. `kubeadm-control-plane.yaml`(defines cluster-api controller node manifest. the `infrastructureRef` field defines the openstack machine template. Find the latest machine template available in the sth cluster and add it to the name field. There is a .yaml file with the name of the image(e.g `osmt-control-plane-g2-2-4-focal-20220610-1-22-10-00-p2.yaml` in our case)  that contains the OpenStackMachineTemplate manifest. The configurations need to be set on this file discussed in the following section) 
```
machineTemplate:
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha4
    kind: OpenStackMachineTemplate
    name: osmt-control-plane-g2-2-4-focal-20220610-1-22-10-00-p2
```
7. `kustomization.yaml`
8. `machine-deployment.yaml`(defines cluster-api worker node manifest, need to set `failureDomain`: `az1`, `az1` is the domain available in sth data center. the `infrastructureRef` field defines the OpenStack machine template. Find the latest machine template available in the sth cluster and add it to the name field. There is a .yaml file with the name of the image(e.g `osmt-g2-2-4-focal-20220610-1-22-10-0.yaml` in our case)  that contains the OpenStackMachineTemplate manifest. the configurations need to be set on this file discussed in the following section 
```
failureDomain: az1
infrastructureRef:
  apiVersion: infrastructure.cluster.x-k8s.io/v1alpha4
  kind: OpenStackMachineTemplate
  name: osmt-g2-2-4-focal-20220610-1-22-10-00
```
9. `Makefile`
10. `openstack-cluster.yaml`(set `nodecidr` and `externalNetworkId` according to the sth related data, these data can be found in another cluster deployed in the sth datacenter, e.g prod-2. set `controlPlaneAvailabilityZones` according to the sth related value `az1`.
```
nodeCidr: 10.227.0.0/18
externalNetworkId: fc91b0c3-3a28-40ce-8781-4802aac218bb # cuspageroonline_sth_standard_ext-net_531
controlPlaneAvailabilityZones:
  - az1
```
11. `osmt-control-plane-g2-2-4-focal-20220610-1-22-10-00-p2.yaml`(defines openstack machine template for the control-plane machine. need to find the image name `soruceUUID` from the openstack image list and add the latest image values here. `serverGroupID` generates from the ops repo via defining the anti-affinity. The generated id can be found in Nordlo sth dashboard)
```
image: ubuntu-focal-server-cloudimg-amd64.img-20220610-k8s-1.22.10-00
rootVolume:
  diskSize: 20
  sourceType: image
  sourceUUID: a8e48fcb-5196-4bc0-aadc-331912730574
securityGroups:
  - name: control-plane
serverGroupID: e1c12b96-32db-498a-b71a-09550cc91e3a
sshKeyName: hawk-deploy-key
```
12. `osmt-g2-2-4-focal-20220610-1-22-10-00.yaml`(defines openstack machine template for the worker node machine. need to find the image name `soruceUUID` from the openstack image list and add the latest image values here. `serverGroupID` generates from the ops repo via defining the anti-affinity. The generated id can be found in Nordlo sth dashboard)
```
image: ubuntu-focal-server-cloudimg-amd64.img-20220610-k8s-1.22.10-00
rootVolume:
  diskSize: 200
  sourceType: image
  sourceUUID: a8e48fcb-5196-4bc0-aadc-331912730574
securityGroups:
  - name: worker
serverGroupID: c7adc9c8-e7b3-4adb-877c-c0780560ffea
sshKeyName: hawk-deploy-key
```

### 1.2. helm-charts

Defines manifests and configuration of the hawk/k8s stuff (e.g argocd, monitor, calico etc).  The argocd contains the k8s manifest to deploy the argocd in Kubernetes. The [repositories config file](https://github.com/pagero/helm-charts/blob/master/umbrellas/argocd/repositories-helm.yaml) which defines all argocd apps repositories of pageroonline(PO) services(k8s/deployments repo in GitBlit) and non-PO/Cluster services(deployments repo in GitHub). When argocd is deployed in a cluster it will sync all PO apps and non-PO/cluster apps into the cluster through these repositories. Following are the main files added to the hlem-charts repo when creating the new k8s cluster(e.g hawk-2). These helm-chart umbrellas are linked to the deployments repo on GitHub. When linking into the deployment repo we need to add the git commit revision(`targetRevision`) where these changes have been made in the helm-chart repo. The best way would be to take the latest commit revision of the master branch which comes after the pull requests are merged with helm-chart changes.

1. `cluster-ingress.yaml`(defines Nginx-ingress configurations of the cluster)
2. `cluster-monitoring`(defines Prometheus, Grafana monitoring of the cluster)
3. `external-dns/hawk-2/kuztomization.yaml`(defines external DNS kuztomization config)
4. `kubernetes-external-secrets/hawk2/kuztomization.yaml`(external secrets of hawk-2 cluster)
5. `openstack-provider/hawk-2/kuztomization.yaml`(defines openstack provider for the cluster)


### 1.3. deployments

Defines argocd apps of the clusters/cluster configurations and some other team services.  Following are the argocd apps defined which are related to the hawk-2 cluster. It contains argocd app of hawk-2 capi-clusters and other services which deploying to the hawk-2 cluster

1. `capi-1/hawk-2`(defines argocd app of hawk-2 capi cluster, this app links to the capi-1 cluster hawk-2 repo. once this app configured in the argocd it will reconcile hawk-2 cluster via cluster-api and capo)
2. `hawk2/hawk/apps`(defines argocd apps of other services which deploying to the hawk-2 cluster. These apps link to the repositories in the helm-chart repo. when liking need to make sure to set `targetRevision` which is the git commit hash of the helm-chart repo where the change has been made.
    1. `cert-manger`(argocd app of hawk-2 cluster cert-manager)
    2. `cni-calico`(argocd app of hawk-2 cluster calico)
    3. `crds-for-prometheus-operator`(argocd app of hawk-2 crds-prometheus-operator)
    4. `cni-cinder`()
    5. `external-dns`
    6. `kubernets-external-secrets`
    7. `kubernets-replicator`
    8. `monitoring`
    9. `nginx-ingress`
    10. `openstack-provider`
    11. `tigera-operator`
```
# helm-charts repository link with targetRevision
source:
  repoURL: git@github.com:pagero/helm-charts.git
  path: umbrellas/cert-manager/base-gcp-serviceaccount
  targetRevision: 1de3aeb228ed71a6eb5c08231a2e47a55b474907
```
3. `hawk-2/hawk/app.yaml`(defines argocd app of apps. in here if the automatic sync policy is enabled, argocd will deploy the app automtically once pull requests are merged to the master branch)
4. `hawk-2/kuztomization.yaml`


### 1.4. ops
Defines terraform cluster setups. build cluster(which runs argocd) and capi-management cluster(which runs cluster api and openstack cluster api operator) deployed via terraform.  build cluster and capi-management cluster runs on gcp. following are the main changes we have done in the ops repo when deploying a new cluster(e.g hawk-2 in sth datacenter). These changes need to manually apply and create server groups vai terraform apply. It will generate two server groups(control-plane and worker-plane). need to find these two serverGroupIDs from Nordlo sth dashboard and add them in the capi-clusters repo as discussed in section
1. `umbrellas/nordlo-sth-openstack/main.tf`(since hawk-2 cluster has been deploying in sth dc, the configurations defined in the Nordlo-sth-openstack. Added server group configurations related to hawk-2 cluster with anti-affinity rules)
```
module "openstack-cluster-hawk-2" {
  source = "../../modules/openstack-cluster"

  cluster_name = "hawk-2"

  providers = {
    openstack = openstack
  }
}

resource "openstack_compute_servergroup_v2" "control-plane-soft-anti-affinity-hawk-2" {
  name     = "hawk-2-control-plane-soft-anti-affinity"
  policies = ["soft-anti-affinity"]
}
```

## 2. Cluster Deployment

### 2.1. Deployment Steps

Following are the steps to deploy the new cluster using argocd and cluster api(with capo)

1. Create pull requests in the above mentioned repositories with the changes
1. Merge pull request of ops repo and apply the changes via terraform. It will give the serverGroupIDs for control-plane and worker-plane. These serverGroupIds can be found in the Nordlo sth dashboard. 
2. Add serverGroupIds in to the capi-clusters repository as mentioned in section 1. Then merge the pull request of capi-clusters repo.
3. Merge pull request of the helm-charts repo and get the latest commit hash(which is the targetRevistion of the deployment repo)
4. Add the latest commit hash of the helm-chart repo as the targetRevistion into the deployments repo as mentioned in section 2. Then merge the pull request of the deployment repo.

### 2.2. Deployment Flow

Once pull requests of the `deployments` repo is merged, the argocd will automatically deploy the cluster. The automatic reconciliation of the cluster happens through the argocd app defined in the `deployment/capi-1/hawk-2/app.yaml app`. Basically, it deploys the cluster through the cluster-api and capo. Once the cluster is deployed, argocd will deploy the specified services into the cluster. This service deployment happens with the argocd apps defined in the `deployments/hawk-2/apps.yaml` repo. The deployment flow of the cluster is described in the following figure.
