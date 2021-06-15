# Module 14 Workshop

## Prerequisites

Before we can spin up a cluster on Azure Kubernetes Service (AKS), we'll first need to install some tools:

### Docker

Follow one of these tutorials to install Docker:
1. [Docker Desktop (Windows)](https://docs.docker.com/docker-for-windows/install/)
2. [Docker Desktop (Mac)](https://docs.docker.com/docker-for-mac/install/)
3. [Docker Engine (Linux)](https://docs.docker.com/engine/install/#server)

Once it's installed, start it up and leave it running in the background.

### The Azure CLI

1. Follow [this tutorial](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) to install the Azure CLI (`az`).
2. Authenticate with your Azure account: open a terminal and run `az login`.
3. Make sure you're running commands against the right subscription: run `az account set --subscription="SUBSCRIPTION_ID"`, replacing `SUBSCRIPTION_ID` with your Softwire Academy subscription ID.

> You can find your subscription ID by logging in to the [Azure Portal](https://portal.azure.com), navigating to the Softwire Academy directory, and opening the Subscriptions service.

### The Kubernetes CLI

Follow [this tutorial](https://kubernetes.io/docs/tasks/tools/) to install the Kubernetes CLI (`kubectl`).

### The Helm CLI

Follow [this tutorial](https://helm.sh/docs/intro/install/) to install the Helm CLI (`helm`).

### A resource group

Every resource that we create today will live inside a resource group, which will be created and given to you at the start of the workshop.

> You may find it helpful to follow along in the [Azure Portal](https://portal.azure.com) as we create each resource.
> You can use the search bar in the Azure Portal to find and view this resource group.

## Installing a Service

In this section we'll create a cluster and use it to run an Nginx server behind a load balancer.

### Spinning up a cluster

Now that we have a resource group, we can create a Kubernetes cluster inside it.
We'll just create a single Node for now; we'll scale up the cluster later in the workshop.

```
az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 1 --generate-ssh-keys
```

> The `az aks create` command can take around 10 minutes to complete.
> You can view the new cluster in the Azure Portal on the `Overview` page of your resource group.

Before we can manage resources on the cluster, we need to get some access credentials.
This command stores the necessary credentials in your `~/.kube/config` folder.
`kubectl` will use these credentials when connecting to the Kubernetes API.

```
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

Let's use `kubectl` to check if our Node is up and running.

```
kubectl get nodes
```

You should see a Node that's `Ready`, e.g.:

```
NAME                     STATUS   ROLES   AGE   VERSION 
aks-default-28776938-0   Ready    agent   5m    v1.19.11
```

> In AKS, Nodes with the same configuration are grouped together into Node pools.
> You can view your Node pool in the Azure Portal on the `Node pools` page of your cluster.
> This Node pool should have a `Node count` of 1, matching the `--node-count` that we specified when creating the cluster.

At this point in the workshop, we've set up our tools, created a resource group, and spun a cluster that contains a single Node.

### Creating some Pods

Now that we have a cluster, let's try deploying an application.
This application will turn our cluster into a basic Nginx server.

Lets start by deploying a Deployment to our cluster.
This Deployment will manage two Pods, each based on the `nginx` container image.
We can create a `module-14-deployment-2-replicas.yaml` file to define our Deployment.

```yaml
# module-14-deployment-2-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: module-14
spec:
  selector:
    matchLabels:
      app: module-14
  replicas: 2
  template:
    metadata:
      labels:
        app: module-14
    spec:
      containers:
        - name: container-name
          image: nginx
          ports:
          - containerPort: 80
```

```
kubectl apply -f module-14-deployment-2-replicas.yaml
```

Now we can watch Pods be created during the deployment.

```
kubectl get pods --watch
```

> You can exit this command by pressing `CTRL+C`.
>
> You should now be able to see a `module-14` Deployment with two available Pods on the `Workloads` page of your cluster.
> If you click through into the `module-14` Deployment's page, then you should see a single ReplicaSet containing two Pods.

### Deploying a Service using `kubectl`

At this point, we have two Pods running on our Node, but they aren't accessible to the outside world.
Let's create a LoadBalancer Service that exposes a single external IP address for the Pods.

```yaml
# service.yaml
kind: Service
apiVersion: v1
metadata:
  name: module-14
spec:
  type: LoadBalancer
  selector:
    app: module-14
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

```
kubectl apply -f service.yaml
```

The LoadBalancer Service type will create an externally accessible endpoint, but this can take a little while to complete.
Let's watch the deployment as it progresses.

```
kubectl get service module-14 --watch
```

Initially, the `EXTERNAL-IP` will be `<pending>`, but after a short while we should get an external IP address.

```
NAME               TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
module-14          LoadBalancer   10.0.37.27   <pending>     80:30572/TCP     6s
```

> You should now see a `module-14` Service on the `Services and ingresses` page of your cluster.
> The Service's `External IP` should match the `EXTERNAL-IP` retrieved above.

We now have a load balancer that will distribute network traffic between our Pods.
Opening the external IP address in a browser should take you to a basic Nginx homepage.

### Changing configuration

If our Service was experiencing heavy load, we might want to increase the number of Pods.
We can manually adjust that by changing the number of replicas in our Deployment definition, then re-deploying it.

```yaml
# module-14-deployment-3-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: module-14
spec:
  selector:
    matchLabels:
      app: module-14
  replicas: 3
  template:
    metadata:
      labels:
        app: module-14
    spec:
      containers:
        - name: container-name
          image: nginx
          ports:
          - containerPort: 80
```

```
kubectl apply -f module-14-deployment-3-replicas.yaml
```

Kubernetes will use the `app: module-14` label to compare what's currently deployed to the cluster (two Pods), with what we want to be deployed (three Pods).
It will then create an additional Pod.

We can check that we now have three Pods.

```
kubectl get pods --watch
```

> The `module-14` Deployment on the `Workloads` page of your cluster should now have three available Pods.

### Removing Deployments and Services

Before we move on to Helm, let's get back to a clean slate by deleting our Deployment and Service.

```
kubectl delete deployment module-14
kubectl delete service module-14
```

We now just have a cluster with a single Node, without any Pods or load balancers.

## Deploying a Service with Helm

So far, we've manually deployed an application using manifests.
We'll now improve this process using Helm.

In this workshop we'll use a local Helm chart.
However, when creating professional applications, you might use a central chart library.

### Deploying a chart

Rather than running `kubectl apply` on each manifest, we can use Helm to deploy them all together:

```
helm install my-chart ./workshop-helm-chart
```

> The install command takes a couple of parameters: the first is the name of the deployment, and the second is the location of the chart.

We can then view the status of the deployment:

```
helm status my-chart
```

Once the deployment has finished, there should be a new `module-14-helm` Service.
Navigating to this Service's IP address should show the Nginx homepage seen earlier.

### Updating a Service

So far, we've only seen Helm doing what `kubectl` can do, so why is it worth the effort?

Let's say we want to apply some specific additional configuration, such as the number of instances required to support the application's load.

Looking in the `values.yaml` file, we can see we've already got a `replicas` field.
We can optionally override that when deploying:

```
helm upgrade --set replicas=4 my-chart ./workshop-helm-chart
```

And then we can watch the new Pod being created:

```
kubectl get pods --watch
```

> Each time we make changes to the Helm chart, we should update the `Chart.yaml` version.
> If we also update the application that the chart points to, we should update `appVersion` too.

## Working with container registries

We'll now have a look at using images from container registries.

### Creating a container registry

We'll start by using the Azure CLI to create a container registry, which will be hosted by Azure.

```
az acr create --resource-group myResourceGroup --name myRegistryName --sku Basic
```

> The `myRegistryName` that you pick must be unique within Azure.
>
> In the Azure Portal, you should now see a container registry on the `Overview` page of your resource group.

This container registry is _private_, and will require credentials to access.

The output of this command should include a `loginServer`.
We'll need this information when pushing to the registry, so make a note of it.

> You can find more information about using container registries in the [Azure documentation](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli).

### Building a container image

Next, we'll build an image that could be added to the registry.

Our image will be based on the Module 13 Workshop application:
1. Copy the contents of the [repository](https://github.com/CorndelWithSoftwire/DevOps-Course-Workshop-Module-13-Learners) into a new `module-13-workshop-application` folder (please do not copy over your code from the previous workshop).
2. Run `cd module-13-workshop-application` to navigate to the folder.
3. Run `docker build -t our-image-name:v1 .` to build an image of the Module 13 Workshop application.
4. Run `cd ..` to move back out to the parent folder.

> If you run `docker image ls` then you should see the newly created image.

### Pushing a container image to a registry

Now that we have an image, we can push it to the registry.

Let's log in to the registry:

```
az acr login --name myRegistryName
```

And then tag the image specifically for the registry:

```
docker tag our-image-name:v1 <login-server>/our-image-name:v1
```

And finally, push the image to the registry:

```
docker push <login-server>/our-image-name:v1
```

> Replace `<loginServer>` with the `loginServer` noted down when creating the registry.
>
> You should now be able to see this image in the Azure Portal on the `Repositories` page of your container registry.

### Using an image from a registry

Our Helm chart currently hardcodes the image in `deployment.yaml`.
Let's make it easier to change dynamically by using a variable instead.

First, add a variable in `values.yaml`, referencing our newly created image, e.g. `image: <loginServer>/our-image-name:v1`.

> `<loginServer>` should be replaced, as before.

Next, update `deployment.yaml` to use this value.
As we're updating the chart, update the `version` in `Chart.yaml` too.

> Helm uses the syntax `{{ .Values.variableName }}` in templates (using Go templates).

You can run `helm template ./workshop-helm-chart` to preview your changes and see what will get deployed.
Once you're happy with the template, try upgrading the chart to use the new image: `helm upgrade my-chart ./workshop-helm-chart`.

### Configuring permissions

If you run `kubectl get pods`, then you'll notice that our Pods aren't able to pull the image that we've just published.
However, we can use a Secret to give our cluster access to the registry, letting the Pods pull the image.

> The syntax for variables and line endings will depend on your terminal.
> The examples shown here work in Bash.

First, let's enable our registry's admin user so we can manage credentials:

```
az acr update -n myRegistryName --admin-enabled true
```

Next, let's retrieve some credentials that can be used to access the registry:

```
LOGIN_SERVER=<loginServer> # `<loginServer>` should be replaced, as before
ACR_USERNAME=$(az acr credential show -n $LOGIN_SERVER --query="username" -o tsv)
ACR_PASSWORD=$(az acr credential show -n $LOGIN_SERVER --query="passwords[0].value" -o tsv)
```

We can then create a Secret with those credentials:

```
kubectl create secret docker-registry acr-secret \
    --docker-server=$LOGIN_SERVER \
    --docker-username=$ACR_USERNAME \
    --docker-password=$ACR_PASSWORD
```
> Note that the multi-line delimiter `\` is for sh-based shells only, for powershell you'll want to use the backtick symbol ` instead

Now that we have a Secret, we can update `deployment.yaml` to use it:

```yaml
spec:
  containers:
    - name: container-name
      image: {{ .Values.image }}
      ports:
      - containerPort: 80
  imagePullSecrets:
    - name: acr-secret
```

Finally, we can update the chart `version` in `Chart.yaml` and upgrade the chart:

```
helm upgrade my-chart ./workshop-helm-chart
```

The Helm chart and its dependencies should now be reachable and deploy correctly.

## Environment variables & secrets

Now that your containers are being deployed correctly we'll want to pass through the appropriate environment variables to link up to the finance package from M13. You can find these environment variables in the configuration from the `order-processing` app service.

> Credentials like DB passwords should be stored as secrets.

## Dealing with Services

We can simulate a high load on our Service by running this code in a browser console on the dashboard from Module 13:

```
await fetch("/scenario", {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({scenario: "VeryHighLoad"})})
```

Let's run `kubectl top pod` to watch the load on the application increase.

Our Pods don't currently have any resource constraints, so the increasing load will eventually use up all of the available resources, taking out our cluster.

Let's prevent that by adding some resource constraints to the Deployment:

```
resources:
  requests:
    memory: "0.5Gi"
    cpu: "500m"
  limits:
    memory: "0.5Gi"
    cpu: "500m"
```

The `requests` fields set the minimum resources available to a Pod, while the `limits` fields set the maximums.

> Kubernetes assigns a [Quality of Service (QoS)](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/) class to each Pod, which reflects how responsive it is likely to be.
> Setting the `requests` and `limits` fields to the same value gives the Pod a `Guaranteed` QoS level.

But what if we want to be able to scale up and handle peaks in demand while staying within resource limits?
We could use a HorizontalPodAutoscaler to automatically spin Pods up and down depending on the application load:

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: {{ .Values.serviceName }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ .Values.serviceName }}
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 80
```

Now if we watch the load on the Node, we'll see more Pods being spun up as the CPU utilisation increases.

Unfortunately, we'll the hit resource limits of our Nodes (5\*500m CPU = 2.5 CPUs), which is more than we've allocated to our Node.
However, we can use a cluster autoscaler to automatically create more Nodes.

```
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 3
```

This should automatically scale up our cluster during high load, which we can watch happen by looking at the Nodes:

```
kubectl get node
```

If we change our Service back to a lower load one, we should see our Service reduce load gradually.
We can watch this using `kubectl get node` and `kubectl get pod`.

```
await fetch("/scenario", {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({scenario: "Initial"})})
```

## Extension exercises

> Kubernetes stores ephemeral logs for Pods, which you can view with the `kubectl logs` command.

We're currently scaling the app automatically, but we aren't monitoring how healthy the app is.
We can improve this by adding some probes, such as a startup probe.

### Startup probes

A startup probe checks whether an application is running.
If the probe notices that the application isn't running then it triggers the parent container's restart policy.

> The default restart policy is `Always`, which causes containers to restart whenever their application exits.
> This applies even if the exit code indicated success.

Startup probes should be declared within the `containers` block of the deployment manifest.
A typical startup probe definition could look like this:

```
startupProbe:
  httpGet:
    path: /health
    port: liveness-port
  failureThreshold: 30
  periodSeconds: 60
```

> This probe could take a long time to check that the app is up and running, so a reduced timeout could be more effective.

We don't have a healthcheck endpoint in our app, so lets add one that just returns 200.
We can then publish our app, along with our helm chart, and watch what happens.

### Resource levels

Under normal load, the memory and CPU we allocated earlier might be much more than needed.
Tune the resource allocations under normal and heavy load and see how the application response to traffic spikes.
You can use the `kubectl top` command to view resource utilisation on Nodes and Pods.

### Incorporating the Finance Package

Up to now we've only been working with one Docker image (for the Order Processing app) and the corresponding pods connect to the Finance Package app deployed on Azure. Let's now try incorporating the Finance Package within the cluster.

> The Docker Hub image for the Finance Package app is available under the name `corndelldevopscourse/mod13-workshop-finance-package:scenarios`

Like with the Order Processing app, you'll need to scrape the environment variables from the app configuration.

You'll also want to setup a service so that the Order Processing app can access the Finance Package app and vice versa.
> Make sure not to expose the Finance Package externally!
### Security

We've so far only used a single service and exposed it fairly crudely, but what if we want a more advanced [ingress](https://docs.microsoft.com/en-us/azure/aks/ingress-basic)?
Try adding some RBAC rules for accessing DB credentials.
You could also add support for encrypting secrets.

## Tidying up

We've finished our adventures in AKS for now, so it's time to delete our resource group.
This will also delete all of its nested resources.

```
az group delete --name myWorkshopResourceGroup
```

> If we were planning to reuse the resource group, then we could just delete the cluster (along with its nested resources) by running `az aks delete --name myAKSCluster --resource-group myResourceGroup`.
