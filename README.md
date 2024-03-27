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

#### create a github PAT for TAP to use

1. generate a PAT in github using the legacy token method. Give the pat access to all repo privileges.
2. copy the `tanzu-cli/values/sensitive-values-template.yml` to `tanzu-cli/values/sensitive-values.yml` and add your newly created PAT and username to it. 
3. create and export the secret using the below commands
```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/secrets/github-pat-template.yml > generated/pat-secret.yaml
tanzu tmc secret create -f generated/pat-secret.yaml -s clustergroup

ytt --data-values-file tanzu-cli/values -f tanzu-cli/secrets/github-pat-export-template.yml > generated/pat-export.yaml
tanzu tmc secret export create -f generated/pat-export.yaml -s clustergroup
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

#### Temporary workaround prior to TAP 1.9

Due to a compatibility issue with TMC provided flux source controller and TAP <=1.8 we need to enable the flux source controller separately from the cluster group level on each cluster. Do the below process on each cluster profile.

```bash
export $PROFILE=<profile-name>
ytt --data-values-file tanzu-cli/values --data-value profile=$PROFILE -f tanzu-cli/cd/source-controller.yml > generated/$PROFILE-sc.yaml
tanzu tmc apply -f generated/$PROFILE-sc.yaml
```

#### Enable at the cluster group level

```
tanzu tmc continuousdelivery enable -g tap-mc -s clustergroup
tanzu tmc helm enable -g tap-vcf -s clustergroup
```






### Downgrade the source controller version

This is a temporary solution that is needed until TAP 1.9, currenltly tmc deploys a newer version of the flux source controller that is incompatible with TAP. Becuase of this we need to downgrade the package version on each cluster.

```
export $PROFILE=<profile-name>
ytt --data-values-file tanzu-cli/values --data-value profile=$PROFILE -f tanzu-cli/cd/source-controller.yml > generated/$PROFILE-sc.yaml
tanzu tmc tanzupackage install update -f generated/$PROFILE-sc.yaml
```