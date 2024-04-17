# TAP on VCF with TMC Guide

This guide's purpose is to quickly stand up TAP using TMC in a VCF environment. This will include dependent resources like DNS and cert management. 


## Tools

* Tanzu CLI
  * TMC plugin
  * Apps Plugin
* yq
* ytt


## Setup the infra 

First we need to create some clusters and setup our intial gitops repo.

### Create the cluster group

This creates the intiial cluster group that all of the tap clusters will be part of and is the base for our gitops setup.

```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cluster-group/cg-template.yml | tanzu tmc clustergroup create -f-
```



### Create cluster group secrets

We can use the TMC cluster group secrets feature to push out opaque or registry credentials to all of out k8s clusters. 

1. copy the `tanzu-cli/values/sensitive-values-template.yml` to `tanzu-cli/values/sensitive-values.yml`

#### create a github PAT for TAP to use

1. generate a PAT in github using the legacy token method. Give the pat access to all repo privileges.
2.  add your newly created PAT and username to the sesnitive valeus file. 
3. create and export the secret using the below commands
```bash
ytt --data-values-file tanzu-cli/values -f tanzu-cli/secrets/github-pat-template.yml > generated/pat-secret.yaml
tanzu tmc secret create -f generated/pat-secret.yaml -s clustergroup

ytt --data-values-file tanzu-cli/values -f tanzu-cli/secrets/github-pat-export-template.yml > generated/pat-export.yaml
tanzu tmc secret export create -f generated/pat-export.yaml -s clustergroup
```

#### Create the AVI credentials secret

1. if you haven't copied the `sensitive-values-template.yml` mentioned in the above section do that step first.
2. update the `avi` section with your credentials
3.  create the cluster group secret

```bash
ytt --data-values-file tanzu-cli/values -f tanzu-cli/secrets/avi-credentials.yml > generated/avi-secret.yaml
tanzu tmc secret create -f generated/avi-secret.yaml -s clustergroup
```


### Create the mutation policies

We are going to create a  mutation policy, these use the gatekeeper mutation policy to inject annotations into different resources.

#### create a policy to enable privileged PSA

This was previosuly done with PSP and TMC by setting the security policy to not enforce PSP and instead use gatekeeper. Now with k8s 1.26+ PSAs are in use. This mutation will label all of the namespaces to effectively turn off PSA so we can use gatekeeper instead.  

```bash
ytt --data-values-file tanzu-cli/values -f tanzu-cli/mutate/psa.yml > generated/psa-mutate.yaml
tanzu tmc policy create -f generated/psa-mutate.yaml -s clustergroup
```
### Create the clusters

This needs to be done for 3 clusters, each with a different profile
* build
* run
* view
  
```bash
export $PROFILE=<profile-name>
ytt --data-values-file tanzu-cli/values --data-value profile=$PROFILE -f tanzu-cli/clusters/cluster-template.yml > generated/$PROFILE-cluster.yaml
tanzu tmc cluster create -f generated/$PROFILE-cluster.yaml
```


### Enable helm and flux
The below commands enable flux at the cluster group level and install the source, helm, and kustomize controllers. These will be installed automatically on all clusters in this cluster group.

#### Enable at the cluster group level

```bash
tanzu tmc continuousdelivery enable -g tap-vcf -s clustergroup
tanzu tmc helm enable -g tap-vcf -s clustergroup
```
### Generate environment specific Gitops files

Some of the flux related files need very specific env related values. This section will generate those files using YTT and the main values file.

#### Generate AKO values

AKO has some very specific environment variables that are needed. this below ytt command generates the kustomize patch file for flux with the correct values from our main values file. 

```bash
ytt --data-values-file tanzu-cli/values -f tanzu-cli/templates/ako-patch-template.yml > flux/apps/clustergroups/tap-vcf/post/ako-patch.yml
```

### Bootstrap cluster with gitops

This step configures the cluster group to use this git repo as the source for flux, specifically the `flux` folder. The gitops setup is done at the cluster group level so we don't need to individually bootstrap every cluster. This allows us to easily install things like cluster issuers,tap overlays, workloads and deliverables. Really it can be used to add anything to your clusters through gitops.  

before creating the TMC objects, you will need to rename the folders in `flux/clusters` to match your cluster names. Also if your cluster group name is different than `tap-vcf` you will need to rename the folder in `flux/clustergroups` along with the paths in the `flux/clustergroups/<group-name>/base.yml`.

create the gitrepo in TMC

```bash
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cd/git-repo-template.yml > generated/gitrepo.yaml
tanzu tmc continuousdelivery gitrepository create -f generated/gitrepo.yaml -s clustergroup
```

create the base kustomization that will bootstrap the clusters and setup any initial infra.


```bash
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cd/kust-template.yml > generated/kust.yaml
tanzu tmc continuousdelivery kustomization create -f generated/kust.yaml -s clustergroup
```

at this point clusters should start syncing in multiple kustomizations. You can check their status using the below command. there will be some in a failed state until the TAP install is done. 

```bash
kubectl get kustomizations -A
```

### Supporting Services

Through the gitops process above we will be installing a few supporting services. These supporting services will allow us to get more public cloud like fucntionality in our VCF environment. Below is the list of what we are configuring and why.

`AKO` -  we will configure AKO in the clusters. In the default TKGs setup that we are using AVI and AKO are already in use for layer 4 networking however it is done in the para virtuaized model. This means that the supervisor cluster is running AKO. For this setup we want to take advantage of a few extra features like automtic DNS and Layer 7 ingress. By installing in the clusters directly we can hand off the AKO responsibilties to the in cluster AKO controller and get access to more features.

`Step-CA`  -  this is a [certificate server](https://smallstep.com/docs/step-ca/installation/#kubernetes) we can install into k8s that is compatibel with cert-manager for generating certs. Rather than using cert manager to generate cluster specific self signed certs we will use this server to grant intermediate issuer authority for our organizations root CA. Read more about the process [here](https://smallstep.com/docs/step-ca/#limitations) and [here](https://smallstep.com/docs/tutorials/intermediate-ca-new-ca/index.html#the-secure-way). 