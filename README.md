# Kubernetes Kustomize Demo

This demo was created to help users learn some basic functionality of Kustomize.  To start, download the project to a working directory on [AWS Cloud9 Workspace](https://console.aws.amazon.com/cloud9/home/product?p=c9&cp=bn&ad=c):

```bash
git clone https://github.com/razavi32/kustomize-demo.git
```

## Explore the Base Directory

Explore the Base directory to understand the kubernetes objects that make up the base application for this demo.  There are two sub-directories, one for each tier of a WordPress application.  Drill into each of the sub-directories and see the components of each tier.  

Run kustomize build to see which resources will be created in each sub-directory:
```bash
cd kustomize-demo/base
cd wordpress
kubectl kustomize build .
cd mysql
kubectl kustomize build .
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
kubectl kustomize build .
```

### Adding a transformer inline

Delete the name-suffix.yml.
```bash
rm name-suffix.yml
```
Run kustomize build to see a preview of the changes.  The suffix should be gone. 

```bash
kubectl kustomize build .
```

Open the kustomization file with vi.

```bash
vi kustomization.yaml
```

Replace the transformer section that was added to the kustomization.yaml with the following:
```bash
transformers:
- |-apiVersion: builtin
    kind: PrefixSuffixTransformer
    metadata:
      name: my-suffix-transformer
    #prefix: v1-
    suffix: -transformers
    fieldSpecs:
    - path: metadata/name
```

Save the kustomization file with vi.

```bash
Esc
:wq!
```

Run kustomize build to see a preview of the changes.  The suffix should be back again. 

```bash
kubectl kustomize build .
```

### Adding a transformer with convenience fields

Open the kustomization file with vi.

```bash
vi kustomization.yaml
```

Replace the transformer section that was added to the kustomization.yaml with the following:
```bash
nameSuffix: -convenience
```

Save the kustomization file with vi.

```bash
Esc
:wq!
```

Run kustomize build to see a preview of the changes.  The suffix should now be convenience. 

```bash
kubectl kustomize build .
```

Deploy the wordpress app (hydrate the kustomization file), check the resources have been created and then delete them.

```bash
kubectl apply -k .
kubectl get all
kubectl delete -k .
```

## Patches

Patches allow you to apply fine-grained changes to resources using JSON or YAML.

### Create a patch.

In this demo, we're going to create a patch that 1) adds an initial container to show the mysql service name and 2) adds an environment variable that allows wordpress to find the mysql database.

Create a patch file with the following content:

```bash
cat <<EOF >./patch.yaml
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
EOF
```

Open the kustomization file with vi.

```bash
vi kustomization.yaml
```

Add the following:
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

Save the kustomization file with vi.

```bash
Esc
:wq!
```

Run kustomize build to see a preview of the changes.  Confirm the variable substitution.  

```bash
kubectl kustomize build .
```

Deploy the Wordpress Application and pull it up in a browser.
```bash
kubectl apply -k .
kubectl get all # Grab the ALB from the response and paste it into a browser. 
kubectl delete -k .
```

### Remove the nameSuffix from the kustomization.yaml.
Open the kustomization file with vi.

```bash
vi kustomization.yaml
```

Remove the following:
```bash
nameSuffix: -convenience
```

Save the kustomization file with vi.

```bash
Esc
:wq!
```
## Multiple Environments w/ Overlays

### Create an Overlays folder

```bash
mkdir ../overlays/dev
```
### Setup the Dev Overlay

Copy the kustomization file from the base directory to the dev overlay.
```bash
cp kustomization.yaml ../overlays/dev
```

Open the kustomization file with vi.

```bash
cd ../overlays/dev
vi kustomization.yaml
```

Replace the current contents with the following:
```bash
namespace: dev

resources:
- "../../base"
- namespace.yml

nameSuffix: -dev
```

Save the kustomization file with vi.

```bash
Esc
:wq!
```

Create the namespace.yml thats referenced above.

```bash
cat <<EOF >./namespace.yml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

Run kustomize build to see a preview of the changes. 

```bash
kubectl kustomize build .
```

Deploy the wordpress app (hydrate the kustomization file), check the resources have been created and then delete them.

```bash
kubectl apply -k .
kubectl get all
kubectl delete -k .
```

### Setup the Prod Overlay

Create a Prod folder under Overlays.
```bash
mkdir ../prod
```

Copy the kustomization file from the dev overlay to the prod overlay.
```bash
cp kustomization.yaml ../prod/
```

Open the kustomization file with vi.

```bash
cd ../overlays/prod
vi kustomization.yaml
```

Replace the current contents with the following:
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

Save the kustomization file with vi.

```bash
Esc
:wq!
```

Create the namespace.yml thats referenced above.

```bash
cat <<EOF >./namespace.yml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```

Run kustomize build to see a preview of the changes. 

```bash
kubectl kustomize build .
```

Deploy the wordpress app (hydrate the kustomization file), check the resources have been created and then delete them.

```bash
kubectl apply -k .
kubectl get all
kubectl delete -k .
```



## [Launching an EKS Cluster - Lab 1](./1-launching-eks-cluster)

Deploy an EKS cluster using [eksctl](https://eksctl.io/).

## [Preparing EKS Cluster for Applications - Lab 2](./2-preparing-eks-cluster )

Execute the following:

* Verify AWS Load Balancer IAM roles for service account is configured
* Deploy AWS Load Balancer Controller for external connectivity
* Deploy official Kubernetes dashboard

## [Containerize Tomcat Web Application - Lab 3](./3-containerize-web-application)

Securely containerize a Java Spring Boot MVC application with Tomcat Servlet for deployment into the cluster. Push containerized application to AWS Elastic Container Registry (ECR).

![airport-data](./3-containerize-web-application/airplane-data/app.png)

## [Deploying Application to EKS Cluster - Lab 4](./4-deploying-application-into-eks)

Deploy the above containerized application from ECR into the newly formed EKS cluster.

## [Update Deployment on EKS Cluster - Lab 5](./5-update-application-deployment)

Update the containerized application by adding additional airports then containerizes a new version for upload to ECR.