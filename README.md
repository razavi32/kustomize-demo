# Kubernetes Kustomize Tutorial

Kustomize helps manage Kubernetes manifests and configurations, making it easier to deploy and manage applications in Kubernetes clusters.  This tutorial was created to help users learn some basic functionality of Kustomize.

## Pre-requisites

The following tutorial assumes you have a running K8s cluster and a workspace where kube config has been setup to talk to that cluster.  To get started quickly, download the project to a working directory on [AWS Cloud9 Workspace](https://console.aws.amazon.com/cloud9/home/product?p=c9&cp=bn&ad=c):

```bash
git clone https://github.com/razavi32/kustomize-demo.git
```

Kustomize comes with kubectl; however, it can be installed stand alone in order to take advantage of the latest features: [Install Kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/).

## Explore the Base Directory

Explore the Base directory to understand the kubernetes objects that make up the base application for this demo.  There are two sub-directories, one for each tier of a WordPress application.  Drill into each of the sub-directories and see the components of each tier.  

Run kustomize build to see which resources will be created in each sub-directory:
```bash
cd kustomize-demo/base
cd wordpress
kubectl kustomize .

cd mysql
kubectl kustomize .
cd ..
```

## Transformers

Transformers are used to modify resources in your Kubernetes manifests before applying them.

Let's say for the sake of simplicity we want to add a suffix to the end of the k8s object names we are about to create.  There are three ways to do that: 1) using a configuration file that's referenced within the kustomization.yaml, 2) adding the transformer inline in the kustomization.yaml or 3) using a convenience field.  Lets walk through each method: 

### Adding a transformer with a configuration file

Create a nameSuffix transformer configuration file in the base directory.

```bash
cat <<EOF >./name-suffix.yml
apiVersion: builtin
kind: PrefixSuffixTransformer
metadata:
  name: my-suffix-transformer
#prefix: v1-
suffix: -transformers
fieldSpecs:
- path: metadata/name
EOF
```

Create a kustomization file for the base directory.

```bash
cat <<EOF >./kustomization.yaml
resources:
- wordpress
- mysql
transformers:
- name-suffix.yml
EOF
```

Run kustomize build to see a preview of the changes:

```bash
kubectl kustomize .
```

### Adding a transformer inline

Delete the name-suffix.yml.
```bash
rm name-suffix.yml
```
Open the kustomization file with vi.  Replace the transformer section that was added to the kustomization.yaml with the following:
```bash
transformers:
- |-
  apiVersion: builtin
  kind: PrefixSuffixTransformer
  metadata:
    name: my-suffix-transformer
  #prefix: v1-
  suffix: -transformers-inline
  fieldSpecs:
  - path: metadata/name
```

Save the kustomization file with vi.  Run kustomize build to see a preview of the changes.  The suffix should be back again but now with -inline tagged onto the enda s well. 

```bash
kubectl kustomize .
```

### Adding a transformer with convenience fields

Open the kustomization file with vi.  Replace the transformer section that was added to the kustomization.yaml with the following:
```bash
nameSuffix: -convenience
```

Save the kustomization file with vi.  Run kustomize build to see a preview of the changes.  The suffix should now be convenience. 

```bash
kubectl kustomize .
```

Deploy the wordpress app (hydrate the kustomization file), check the resources have been created and then delete them.

```bash
kubectl apply -k .

kubectl get all

kubectl delete -k .
```

## Generators

Generators dynamically generate Kubernetes resources (ConfigMaps and Secrets) based on configurations at the time of applying your Kustomize manifests.

In this demo, we're going to add a Secrets Generator to our Kustomization yaml in the mysql directory.  This generator is going to be the root password to our mysql database.  As with transformers, there are multiple methods for adding a Generator.  In this example, we're going to use environment variables.  

Create a properties file with a password key/value pair.  
```bash
cd mysql

cat <<EOF >./mysql-secret.properties
password=admin
EOF
```

Open the kustomization file in the mysql directory with vi.  Add the following to the top of the file:
```bash
secretGenerator:
  - name: mysql-pass
    envs:
      - mysql-secret.properties
```
Save the kustomization file with vi.  Run kustomize build to see a preview of the changes.  There should be a secret generated first and notice the deployment references the right name for the secret. 

```bash
kubectl kustomize .
```

## Patches

Patches allow you to apply fine-grained changes to resources using JSON or YAML.

### Create a patch

In this demo, we're going to create a patch that 1) adds an initial container to show the mysql service name and 2) adds an environment variable that allows wordpress to find the mysql database.

Create a file called patch.yaml in the base directory with the following content. 

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  template:
    spec:
      initContainers:
      - name: init-command
        image: debian
        command: ["/bin/sh"]
        args: ["-c", "echo $(WORDPRESS_SERVICE); echo $(MYSQL_SERVICE)"]
      containers:
      - name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: $(MYSQL_SERVICE)
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
```

In addition to the patch, we also need to bind variables to K8s object fields.  This can be done with variable substitution.  In other words, regardless of the actual service names, the wordpress application will be able to find the mysql service.  Open the kustomization file with vi.  Add the following:
```bash
patches:
- path: patch.yaml

vars:
  - name: WORDPRESS_SERVICE
    objref:
      kind: Service
      name: wordpress
      apiVersion: v1
    fieldref:
      fieldpath: metadata.name
  - name: MYSQL_SERVICE
    objref:
      kind: Service
      name: mysql
      apiVersion: v1
```

Save the kustomization file with vi.  Run kustomize build to see a preview of the changes.  Confirm the variable substitution.  

```bash
kubectl kustomize .
```

Deploy the Wordpress Application and pull it up in a browser.  Once you're done reviewing the application, delete all the resources.  
```bash
kubectl apply -k .

kubectl get all | grep LoadBalancer # Grab the ALB from the response and paste it into a browser. 

kubectl delete -k .
```

### Remove the nameSuffix from the kustomization.yaml.
Open the kustomization file with vi.  Remove the following:
```bash
nameSuffix: -convenience
```
Save the kustomization file with vi.

## Multiple Environments w/ Overlays

Kustomize can be used to deploy manifests to different environments like dev, test and production by leveraging overlays.  Our base manifests (base directory) represent our application with a common configuration that is shared across all environments.  In this section, we'll create separate overlay directories for a dev and prod environment and then customize the base manifests with environment-specific configurations or settings.

### Create an Overlays folder

```bash
mkdir ../overlays
mkdir ../overlays/dev
```
### Setup the Dev Overlay

Create a Kustomization file in the dev overlay directory. 
```bash
cd ../overlays/dev
vi kustomization.yaml
```

Add the following contents to the file:
```bash
namespace: dev

resources:
- "../../base"
- namespace.yml

images:
- name: wordpress
  newTag: 5.0-apache

nameSuffix: -dev
```

Save the kustomization file with vi.  Create the namespace.yml thats referenced above.

```bash
cat <<EOF >./namespace.yml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
EOF
```

Run kustomize build to see a preview of the changes. 

```bash
kubectl kustomize .
```

Deploy the wordpress app (hydrate the kustomization file), check the resources have been created and then delete them.

```bash
kubectl apply -k .

kubectl get all -n dev

kubectl delete -k .
```

### Setup the Prod Overlay

Create a Prod folder under Overlays.
```bash
mkdir ../prod
```

Create a Kustomization file in the prod overlay directory. 
```bash
cd ../prod
vi kustomization.yaml
```

Add the following contents to the file:
```bash
namespace: prod

resources:
- "../../base"
- namespace.yml

nameSuffix: -prod

labels:
- pairs:
    env: prod

replicas:
- name: wordpress
  count: 3
```

Save the kustomization file with vi.  Create the namespace.yml thats referenced above.

```bash
cat <<EOF >./namespace.yml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
EOF
```

Run kustomize build to see a preview of the changes. 

```bash
kubectl kustomize .
```

Deploy the wordpress app (hydrate the kustomization file), check the resources have been created and the app is up.

```bash
kubectl apply -k .

kubectl get all -n prod | grep LoadBalancer # Grab the ALB from the response and paste it into a browser. 
```

Delete the resources.
```bash
kubectl delete -k .
```

## Integrating Kustomize with CI/CD

Kustomize can be effectively integrated into a CI/CD (Continuous Integration/Continuous Deployment) pipeline alongside your existing CI/CD tools and workflows. For example, you can use Kustomize to generate Kubernetes manifests as part of your pipeline, apply changes to your Kubernetes clusters using kubectl, and automate testing and validation steps to ensure smooth deployments.  In this tutorial, we're going to use a GitHub Actions workflow to demonstrate pushing code to a repo that kicks off an automated workflow.  The workflow will go through a series of steps which include deploying resources to a K8s cluster. 