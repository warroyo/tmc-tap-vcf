# TAP on VCF with TMC Guide

This guide's purpose is to quickly stand up TAP using TMC in a VCF environment. This will include dependent resources like DNS and cert management. 

## What it does

Here is a list of the things that this project does

* install AKO in all clusters
* installs [step-ca](https://smallstep.com/docs/step-ca/) in view cluster to be a centralized self hosted intermdiate acme server for a enterprise root
* installs TAP using TMC
* configures automatic DNS on all TAP components with ingress using AVI DNS
* configures cert manager to use the self hosted acme server
* generates all certs automatically using the self hosted acme server
* handles acme challenges with http01 and auto dns from AVI
* sets up Flux for gitops of everything
* uses TMC opaque secrets as a simple secret provider
* creates a custom workload type to take advantage of layer 7 ingress with AVI
* deploys a sample workload using automated certs and dns provided by the above features

## Pre-reqs

* AVI configured with a DNS VS, [docs](https://avinetworks.com/docs/latest/avi-dns-architecture/#authoritative-name-server-for-a-subdomain-zone)
* TKGs deployed with AVI + NSXT

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

```bash
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

#### Create the registry creds secret
1. update the `registry` section with your credentials in the sensitive values
2.  create the cluster group secret

```bash
ytt --data-values-file tanzu-cli/values -f tanzu-cli/secrets/registry-creds.yml > generated/registry-creds.yaml
tanzu tmc secret create -f generated/registry-creds.yaml -s clustergroup
```
3. export the secret

```bash
ytt --data-values-file tanzu-cli/values -f tanzu-cli/secrets/registry-export.yml > generated/registry-export.yaml
tanzu tmc secret export create -f generated/registry-export.yaml -s clustergroup
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


### Supporting Services

Through the gitops process above we will be installing a few supporting services. These supporting services will allow us to get more public cloud like fucntionality in our VCF environment. Below is the list of what we are configuring and why.

`AKO` -  we will configure AKO in the clusters. In the default TKGs setup that we are using AVI and AKO are already in use for layer 4 networking however it is done in the para virtuaized model. This means that the supervisor cluster is running AKO. For this setup we want to take advantage of a few extra features like automtic DNS and Layer 7 ingress. By installing in the clusters directly we can hand off the AKO responsibilties to the in cluster AKO controller and get access to more features.

`Step-CA`  -  this is a [certificate server](https://smallstep.com/docs/step-ca/installation/#kubernetes) we can install into k8s that is compatibel with cert-manager for generating certs. Rather than using cert manager to generate cluster specific self signed certs we will use this server to grant intermediate issuer authority for our organizations root CA. Read more about the process [here](https://smallstep.com/docs/step-ca/#limitations) and [here](https://smallstep.com/docs/tutorials/intermediate-ca-new-ca/index.html#the-secure-way). 


### setting up the root CA

In this example we will mimic creating an intermediate CA that step CA will use to sign and issue certs. In an enterprise environment usually there is already an existing CA, this process will be used to securely authorize step-ca to create certs on it's behalf. All of the details can be found [here](https://smallstep.com/docs/tutorials/intermediate-ca-new-ca/). You can skip the first piece of creating the intial CA if you already have a corporate CA.

#### Generate intial CA to mimic enterpise CA

**if you already have a CA skip this step**

Be sure to save any generated passwords in these steps!

1. run the below command and let it generate a password for you. This will generate our fake root cert to mimic an enterpise cert.
```bash
export STEPPATH=./companyroot
step ca init  --deployment-type='standalone' --name='companyroot' --dns='localhost' --address='127.0.0.1:8443' --provisioner='admin@company.com' 
```

The above command has created our mock enterpsie CA

2. we need to generate some default things before using our enterpise root ca. This step generates those defaults as defined in the docs above. 
```bash
export STEPPATH=./intermediateca
step ca init  --deployment-type='standalone' --name='intermediateroot' --dns='step-certificates.step-ca.svc.cluster.local' --address=':9000' --provisioner='acme' --acme
```

3. swap out the default root ca with out "companyroot" ca

```bash
rm intermediateca/secrets/root_ca_key
cp companyroot/certs/root_ca.crt  intermediateca/certs/root_ca.crt
```

4. generate a new signing key and intermediate certificate signed by our mock compnay root ca.
```bash
step certificate create "company k8s intermediate" intermediate.csr intermediate_ca_key --csr
```

5. sign the intermediate with the companyroot ca
```bash
step certificate sign --profile intermediate-ca intermediate.csr companyroot/certs/root_ca.crt companyroot/secrets/root_ca_key > intermediate.crt
```

6. replace the default intermediates with the newly signed ones.

```bash
mv intermediate.crt intermediateca/certs/intermediate_ca.crt
mv intermediate_ca_key intermediateca/secrets/intermediate_ca_key
rm intermediate.csr
```

#### Create an intermediate from the enterprise root CA

**if you are using the above mock approach skip this step**

We will be using the ["secure way"](https://smallstep.com/docs/tutorials/intermediate-ca-new-ca/#the-secure-way) laid out in the step-ca docs.

1. follow the steps in the doc above.

#### Generate necessary helm values


1. add your provisioner and decrypt secrets to the sensitive values file. This will be the provisioner password and the ca password that should have been set during the previosu step. These are added under the `steps-ca` section.



2. create the intermediate ca key secret

```bash
ytt  --data-values-file tanzu-cli/values -f tanzu-cli/secrets/step-ca-secrets.yml -f intermediateca/. > generated/intermediate-ca-key.yml
tanzu tmc secret create -f generated/intermediate-ca-key.yml -s clustergroup

```


3. create the secret that hold the certs

```bash
ytt  --data-values-file tanzu-cli/values -f tanzu-cli/secrets/step-ca-certs.yml -f intermediateca/. > generated/intermediate-ca-certs.yml
tanzu tmc secret create -f generated/intermediate-ca-certs.yml -s clustergroup
```

4. create the secret that holds the configs

```bash
ytt  --data-values-file tanzu-cli/values -f tanzu-cli/secrets/step-ca-config.yml -f intermediateca/. > generated/intermediate-ca-config.yml
tanzu tmc secret create -f generated/intermediate-ca-config.yml -s clustergroup
```
5. create the secret that hold the ca decrypt password. this was the password set when we ran the step-ca init commands above.

```bash
ytt  --data-values-file tanzu-cli/values -f tanzu-cli/secrets/step-ca-password.yml -f intermediateca/. > generated/intermediate-ca-password.yml
tanzu tmc secret create -f generated/intermediate-ca-password.yml -s clustergroup
```

6. create the secret that hold the provisioner decrypt password. this was the password set when we ran the step-ca init commands above.

```bash
ytt  --data-values-file tanzu-cli/values -f tanzu-cli/secrets/step-ca-prov-password.yml -f intermediateca/. > generated/intermediate-ca-prov-password.yml
tanzu tmc secret create -f generated/intermediate-ca-prov-password.yml -s clustergroup
```

7. create the issuer config secret that flux will use to populate values in the issuer

```bash
ytt  --data-values-file tanzu-cli/values -f tanzu-cli/secrets/step-ca-issuer.yml -f intermediateca/. > generated/intermediate-ca-issuer-config.yml
tanzu tmc secret create -f generated/intermediate-ca-issuer-config.yml -s clustergroup
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

### Create the static DNS records

Becuase we are using AVI for DNS, there is a step required to add some CNAME records into AVI. This is becuase the AVI DNS integration with AKO does not support wildcards. To get around this we will use the supported AKO L$ dns integration to create a dynamic A record and then create a static CNAME record that is a wildcard. Thsi way if the IP changes, AKO will update it and we don't need to manually update records.


1. get the dns vs uuid

```bash
export VS_NAME='<your dns vs here>'
export VS_UUID=$(curl -k -H "Content-Type: application/json" -H "x-avi-version: 22.1.5"  -XGET -u 'admin:VMware1!' "https://10.214.181.32/api/virtualservice?name=$VS_NAME" | jq -r '.results[0].uuid')
```


2. create the records


```bash
curl -k -H "Content-Type: application/json" -H "x-avi-version: 22.1.5"  -XGET -u 'admin:VMware1!' https://10.214.181.32/api/virtualservice/$VS_UUID | ytt --data-values-file tanzu-cli/values  -f tanzu-cli/templates/cname-json.yml -f- -o json | curl -k -H "Content-Type: application/json" -H "x-avi-version: 22.1.5"  -X PUT -u 'admin:VMware1!' --json @- https://10.214.181.32/api/virtualservice/$VS_UUID
```

## Install TAP


### Add the cluster url and ca to the values file

We need to pull back this info from the cli and update our values file. This is used so the view cluster can connect to the other clusters. 

run this for build and run profiles

**Build:**

```
export PROFILE=build
export MGMT_CLUSTER=$(cat tanzu-cli/values/values.yml | yq .clusters.$PROFILE.mgmt_cluster)
export CLUSTER_NAME=$(cat tanzu-cli/values/values.yml | yq .clusters.$PROFILE.name)
export PROVISIONER=$(cat tanzu-cli/values/values.yml | yq .clusters.$PROFILE.provisioner)
tanzu tmc cluster kubeconfig get $CLUSTER_NAME -m $MGMT_CLUSTER -p $PROVISIONER | ytt --data-values-file - --data-value profile=$PROFILE -f tanzu-cli/overlays/clusterdetails.yml -f tanzu-cli/values/values.yml --output-files tanzu-cli/values
```

**Run:** 

```
export PROFILE=run
export MGMT_CLUSTER=$(cat tanzu-cli/values/values.yml | yq .clusters.$PROFILE.mgmt_cluster)
export CLUSTER_NAME=$(cat tanzu-cli/values/values.yml | yq .clusters.$PROFILE.name)
export PROVISIONER=$(cat tanzu-cli/values/values.yml | yq .clusters.$PROFILE.provisioner)
tanzu tmc cluster kubeconfig get $CLUSTER_NAME -m $MGMT_CLUSTER -p $PROVISIONER | ytt --data-values-file - --data-value profile=$PROFILE -f tanzu-cli/overlays/clusterdetails.yml -f tanzu-cli/values/values.yml --output-files tanzu-cli/values
```


### Create the TAP solution

The below command with generate our TMC TAP install file from the values and then create the solution. This will start the install across all clusters. you can check the status in the TMC UI.

```
export TAP_NAME=$(cat tanzu-cli/values/values.yml| yq .tap.name)
ytt --data-values-file tanzu-cli/values -f tanzu-cli/tap/tap-template.yml > generated/tap.yaml
tanzu tmc tanzupackage tap create -n $TAP_NAME -f generated/tap.yaml
```


### Update the TAP solution

You can change the values in the values.yml and update the solution through the cli as well.

```bash
export TAP_NAME=$(cat tanzu-cli/values/values.yml| yq .tap.name)
tanzu tmc tanzupackage tap get -n $TAP_NAME -o yaml | sed '1d' | ytt --data-values-file - --data-values-file tanzu-cli/values -f tanzu-cli/overlays/generation.yml -f intermediateca/. -f tanzu-cli/tap/tap-template.yml > generated/tap.yaml
tanzu tmc tanzupackage tap update -n $TAP_NAME -f generated/tap.yaml
```