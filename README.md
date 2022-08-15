# TAP Supply Chain to deploy a helm chart using FluxCD

## Why do this
One of the great features in TAP is the UI provided which is based on Backstage.  
Backstage and in accordance, TAP GUI includes a plugin called Techdocs that allows us to have technical documentation rendered in the UI for us that is written in Markdown.  
One of the challenges is where and when the rendering of the Markdown into a static HTML site happens.  
Backstage supports multiple different options and I have blogged and written a YTT overlay to make auto rendering possible by adding a sidecar container to TAP GUI that has a docker socket within it and then using the docker based auto rendering of the docs.  
While that approach works, it is custom, less secure, resource intensive and has some edge cases it doesnt handle as well.  
The supported and documented approach by VMware in TAP is to pre render the docs and store them in an S3 bucket, and TAP GUI will simply pull down the files from the bucket and render the HTML files inside the UI.  
To generate the Techdocs client side is a pain and requires npm which is always a "unique" experience.  
If we could have cartographer via a tekton task automate this process so that at every commit the docs were re-generated it would be amazing!

## How to do this

### Overview
The goal here is to create a Tekton task that uses a custom docker image I have built using the Dockerfile in this repo which contains mkdocs, plantuml and techdocs-cli along with the needed dependencies, in order to auto generate the docs as part of a supply chain and to push them to an S3 bucket.

#### DISCLAIMER - I have tested this with AWS S3 and Minio only. other providers may work but have not been tested.

### Creating the new Tekton task
The first thing we need to do for this supply chain is to install the custom Tekton task that will generate and publish our techdocs as part of our supply chain execution.

```bash
kubectl apply -f templates/cluster-task.yaml
```

### Create the custom Cartographer templates

The next step is to create the cartographer resources to allow us to utilize the new Tekton task, in our platform.

```bash
kubectl apply -f templates/crt.yaml
kubectl apply -f templates/cst.yaml
```

### Creating the custom supply chain
The first step is to fill in the values.yaml file in this repo based on your environment. These can be the same values as you have supplied for the ootb_supply_chain section in your tap installation values.

Once you have set the values in the file, we can now render and apply the supply chain manifest:

```bash
ytt -f values.yaml -f supply-chain.yaml | kubectl apply -f -
```

### Create the S3 credentials secret
The final step in preparing the environment is to create a secret with the details of how to connect to our S3 bucket, with credentials high enough to push the new docs to the bucket. for exact permissions check the TAP GUI or backstage documentation.  
The kubernetes secret should be created based on the following example
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: techdocs-s3-creds
type: Opaque
stringData:
  access-key: FILL-ME-IN
  bucket-name: FILL-ME-IN
  endpoint-url: FILL-ME-IN
  region: FILL-ME-IN
  secret-key: FILL-ME-IN
```  
Once we have the secret created in our workloads namespace, we can deploy a workload and test the new supply chain out!  

### Testing out the new supply chain
To test out the new supply chain, we can use the example workload in this repo
```bash
tanzu apps workload apply -f example/workload.yaml
