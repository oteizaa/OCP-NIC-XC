# OpenShift Clusters with NGINX Ingress Controller, and high availability using F5 XC

Ensure to have the Azure CLI 2.6.0 or later to create the OpenShift Clusters in Azure (ARO)

   ```bash
   az --version
   ```

## Register the resource providers

1) If you have multiple Azure subscriptions, specify the relevant subscription ID:

   ```bash
   az account set --subscription <SUBSCRIPTION ID>
   ```

2) Register the Microsoft.RedHatOpenShift resource provider:

   ```bash
   az provider register -n Microsoft.RedHatOpenShift --wait
   ```

3) Register the Microsoft.Compute resource provider:

   ```bash
   az provider register -n Microsoft.Compute --wait
   ```

4) Register the Microsoft.Storage resource provider:

   ```bash
   az provider register -n Microsoft.Storage --wait
   ```

5) Register the Microsoft.Authorization resource provider:

   ```bash
   az provider register -n Microsoft.Authorization --wait
   ```

6) ARO preview features are available on a self-service, opt-in basis. Preview features are provided "as is" and "as available," and they are excluded from the service-level agreements and limited warranty. Preview features are partially covered by customer support on a best-effort basis. As such, these features are not meant for production use.

   ```bash
   az feature register --namespace Microsoft.RedHatOpenShift --name preview
   ```

## Create a virtual network containing two empty subnets

1) Set the following variables in the shell environment in which you will execute the az commands.

   ```bash
   LOCATION=eastus                 # the location of your cluster
   RESOURCEGROUP=aro-rg            # the name of the resource group where you want to create your cluster
   CLUSTER=cluster                 # the name of your cluster
   ```

2) Create a resource group.

   ```bash
   az group create \
     --name $RESOURCEGROUP \
     --location $LOCATION
   ```

3) Create a virtual network.

   ```bash
   az network vnet create \
      --resource-group $RESOURCEGROUP \
      --name aro-vnet \
      --address-prefixes 10.0.0.0/22
   ```

4) Add an empty subnet for the master nodes.

   ```bash
   az network vnet subnet create \
     --resource-group $RESOURCEGROUP \
     --vnet-name aro-vnet \
     --name master-subnet \
     --address-prefixes 10.0.0.0/23 \
     --service-endpoints Microsoft.ContainerRegistry
   ```

5) Add an empty subnet for the worker nodes.

   ```bash
   az network vnet subnet create \
     --resource-group $RESOURCEGROUP \
     --vnet-name aro-vnet \
     --name worker-subnet \
     --address-prefixes 10.0.2.0/23 \
     --service-endpoints Microsoft.ContainerRegistry
   ```

6) Disable subnet private endpoint policies on the master subnet. This is required for the service to be able to connect to and manage the cluster.

   ```bash
   az network vnet subnet update \
     --name master-subnet \
     --resource-group $RESOURCEGROUP \
     --vnet-name aro-vnet \
     --disable-private-link-service-network-policies true
   ```

## Create the OpenShift cluster

   ```bash
   az aro create \
     --resource-group $RESOURCEGROUP \
     --name $CLUSTER \
     --vnet aro-vnet \
     --master-subnet master-subnet \
     --worker-subnet worker-subnet
   ```
## Get your OpenShift credentials

   ```bash
   az aro list-credentials --name $CLUSTER --resource-group $RESOURCEGROUP
   ```
## Access Openshift

Install oc in your laptop, following this link [Installing OC CLI on Linux](https://docs.openshift.com/container-platform/4.2/cli_reference/openshift_cli/getting-started-cli.html#cli-installing-cli-on-linux_cli-developer-commands)

Take note of your OpenShift Console url and API server URL

Access yo the following URL to request a new token:

1. https://< YOUR CONSOLE URL>/oauth/token/request

Provide your username (kubeadmin) and your password

Click on "Display token". Now you can access from your CLI using the following script:

   ```bash
   oc login --token=<YOUR TOKEN> --server=<YOUR API SERVER URL>
   ```

Test you are accessing OK you your OpenShift enviroment (for this lab we are going to use kubectl)

   ```bash
   kubectl get nodes
   ```
You will get something like this:

   ```shell
   NAME                                 STATUS   ROLES    AGE   VERSION
   cluster-btqgx-master-0               Ready    master   15h   v1.23.5+3afdacb
   cluster-btqgx-master-1               Ready    master   15h   v1.23.5+3afdacb
   cluster-btqgx-master-2               Ready    master   15h   v1.23.5+3afdacb
   cluster-btqgx-worker-eastus1-48fpj   Ready    worker   15h   v1.23.5+3afdacb
   cluster-btqgx-worker-eastus2-xgxl2   Ready    worker   15h   v1.23.5+3afdacb
   cluster-btqgx-worker-eastus3-l7tb8   Ready    worker   15h   v1.23.5+3afdacb
   ```

After you finish this guide, repeat to your second Openshift cluster


# Deploy the Microservices App in each OCP Cluster

## Fork the workshop repository

1. To proceed with this scenario, you will need to fork the workshop repository to your GitHub account.  If this is your first time, then take a few minutes to review the [GitHub Docs on how to Fork a repo](https://docs.github.com/en/get-started/quickstart/fork-a-repo).

    You can complete this task through the [repository GitHub UI](https://github.com/f5devcentral/modern_app_jumpstart_workshop):
    ![GitHub Fork](../assets/gh_fork.jpg)

    or via the GitHub CLI:

    ```bash
    gh repo fork --clone f5devcentral/modern_app_jumpstart_workshop
    ```

## Clone your workshop repository to your laptop

Now that you have forked the workshop repository, you'll want to clone the repo to your local laptop.  

1. Perform this via the git or GitHub CLI commands.

    > **Note:** Make sure to replace your_username with your GitHub username.

    > **Note:** If you have not [configured GitHub authentication](https://docs.github.com/en/authentication) with your local laptop, please stop and do that now.

    ```bash
    # via HTTPS
    git clone https://github.com/your_username/modern_app_jumpstart_workshop.git modern_app_jumpstart_workshop

    # via SSH
    git clone git@github.com:your_username/modern_app_jumpstart_workshop.git modern_app_jumpstart_workshop
    ```

# Argo CD

[Argo CD](https://argoproj.github.io/cd/) is a declarative, GitOps continuous delivery tool for Kubernetes.

In our workshop, we will use Argo CD to deploy our microservices and resources.

## Install Argo CD

1. On your laptop run:

    ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

    ```

## Expose the Argo CD Server API/UI

1. Use OpenShift Route to expose ArgoCD server

We need to patch the ArgoCD Server deployment on OpenShift so that the service is exposed via OpenShift routing:

   ```bash
   oc -n argocd patch deployment argocd-server -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"argocd-server"}],"containers":[{"command":["argocd-server","--insecure","--staticassets","/shared/app"],"name":"argocd-server"}]}}}}'
   ```

You should get Tinker Whether it was successful in the output.

You can then continue to expose the ArgoCD server:

   ```bash
   oc -n argocd create route edge argocd-server --service=argocd-server --port=http --insecure-policy=Redirect
   ```

Confirm that the route has been created, you will get a message like this one:

   ```shell
   oc get route -n argocd
   NAME            HOST/PORT                                             PATH   SERVICES        PORT   TERMINATION     WILDCARD
   argocd-server   argocd-server-argocd.apps.osg8bic8.eastus.aroapp.io          argocd-server   http   edge/Redirect   None
   ```

Access to your ARGOCD server

## Login to Argo CD

1. Obtain the Argo CD password:

    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
    ```

1. Use the Argo CD UDF Access Method to access the Argo CD UI and login with the `admin` user and the password you obtained in the previous step.


# Build NGINX Plus Ingress Controller Container

In this step, you will build a copy of the NGINX Plus Ingress Controller container and push it to your private container registry.

**Note:** We **HIGHLY** discourage you from publishing your NGINX Plus containers to a public registry. Please ensure you are publishing the containers from this lab to a private registry such as [GitHub Packages](https://github.com/features/packages) or [Docker Hub](https://hub.docker.com/).

You can also reference the official [NGINX Ingress Controller documentation](https://docs.nginx.com/nginx-ingress-controller/) for additional details.

## Clone the NGINX Ingress Controller Repository

1. In your terminal, clone the Official [NGINX Ingress Controller repository](https://github.com/nginxinc/kubernetes-ingress.git)

    **Note:** You may need to update the branch version to match the latest release of NGINX Ingress Controller

    ```bash
    git clone https://github.com/nginxinc/kubernetes-ingress.git --branch v2.3.0
    cd kubernetes-ingress
    ```

## Build the Container

For this step, we will leverage the [Docker CLI](https://docs.docker.com/engine/install/) to build the NGINX Ingress Controller image. Alternatively, you can use [Podman](https://podman.io/) if you do not have a Docker license.

The repository's Makefile supports several [target types](https://docs.nginx.com/nginx-ingress-controller/installation/building-ingress-controller-image/#makefile-targets), but for this lab we will leverage the *debian-image-nap-dos-plus* target so we can use NGINX App Protect WAF.

> **Note:** For additional details you can also reference the [Build the Ingress Controller Image](https://docs.nginx.com/nginx-ingress-controller/installation/building-ingress-controller-image/) portion of the [NGINX Ingress Controller documentation](https://docs.nginx.com/nginx-ingress-controller/).

1. Make sure that the certificate (nginx-repo.crt) and the key (nginx-repo.key) of your license are located in the root of the project:

    ```bash
    ls nginx-repo.*
    nginx-repo.crt  nginx-repo.key
    ```

1. To build the NGINX Ingress Controller container, follow these steps:

    ```bash
    # Replace OWNER with your Github username
    export GITHUB_USER=OWNER
    make debian-image-nap-dos-plus PREFIX=ghcr.io/$GITHUB_USER/nginx-plus-ingress TARGET=container DOCKER_BUILD_OPTIONS="--platform linux/amd64"
    ```

## Publish the Container

To publish the NGINX Ingress Controller container to your private registry follow the following steps:

1. Create a [GitHub PAT (Personal Access Token)](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) with the following scopes:
    - *read:packages*
    - *write:packages*
    - *delete:packages*

1. Export the value to the *GITHUB_TOKEN* environment variable.

    ```bash
    export GITHUB_TOKEN=your_access_token
    ```

1. Run the *docker login* command to log into the [GitHub Package](https://github.com/features/packages) container registry with your PAT:

    ```bash
    # Login to GitHub Packages
    echo $GITHUB_TOKEN | docker login ghcr.io -u $GITHUB_USER --password-stdin

    # Find your container tag
    TAG=`docker images ghcr.io/$GITHUB_USER/nginx-plus-ingress --format "{{.Tag}}"`

    # Publish the container
    docker push ghcr.io/$GITHUB_USER/nginx-plus-ingress:$TAG
    ```

    > **Note:** If you had previously created and tagged an `nginx-plus-ingress` container image on your system, the command above used to set the `TAG` variable will not work. Instead, run `docker images ghcr.io/$GITHUB_USER/nginx-plus-ingress --format "{{.Tag}}"` and select your most recent tag from the output, then set the variable manually: `TAG=<your tag from the previous command>`.




# Install NGINX Plus Ingress

For this step, we will pull the NGINX Plus Ingress Controller image from your private registry and deploy it into your K3s deployment.

We will use Argo CD to deploy NGINX Ingress Controller for us. However, if you wanted to do this using the Helm CLI, you may use [this procedure](install_nic_helm.md) as a reference.

Alternatively, if you wish to install the NGINX Ingress Controller from the NGINX private container registry, you can follow [this procedure](install_nic_nginx_registry.md) and skip the remainder of this document.

> **Note:** For more details, you can access the NGINX Ingress Controller documentation [here](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/)

## Create a Read-Only GitHub PAT (Personal Access Token)

While you could leverage the PAT created in the build steps, the best practice is to leverage a least privilege model and create a read-only PAT for your Kubernetes cluster.

1. Create a [GitHub PAT (Personal Access Token)](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) with the following scopes:
    - *read:packages*

1. Export the value to the *GITHUB_TOKEN* environment variable.

    ```bash
    export GITHUB_TOKEN=your_access_token
    ```

## Create Kubernetes Secret

1. In order to pull the NGINX Plus Ingress container from your private registry, the K8s cluster will need to have access to the GitHub PAT you created previously. To accomplish this, you will create a docker-registry secret with the commands below:

    ```bash
    # Create nginx-ingress namespace
    kubectl create namespace nginx-ingress

    export GITHUB_USER=your_github_username

    # create container registry secret
    kubectl create secret docker-registry ghcr -n nginx-ingress --docker-server=ghcr.io --docker-username=${GITHUB_USER} --docker-password=${GITHUB_TOKEN}
    ```

## Update Helm Values and Argo CD Application Manifest

Before you can deploy the NGINX Ingress Controller, you will need to modify the Helm chart values to match your environment.

1. Find your NGINX Plus Ingress Controller container tag with the following command:

    ```bash
    TAG=`docker images ghcr.io/$GITHUB_USER/nginx-plus-ingress --format "{{.Tag}}"`
    echo $TAG
    ```

    > **Note:** If you had previously created and tagged an `nginx-plus-ingress` container image on your system, the command above used to set the `TAG` variable will not work. Instead, run `docker images ghcr.io/$GITHUB_USER/nginx-plus-ingress --format "{{.Tag}}"` and select your most recent tag from the output, then set the variable manually: `TAG=<your tag from the previous command>`.

1. Open the `charts/nginx-plus-ingress/values.yaml` file in your forked version of the **infra** repository.

1. Find the following variables and replace them with your information:

    | Variable        | Value           |
    |-----------------|-----------------|
    | \<GITHUB_USER\>   | github username |
    | &lt;TAG>        | tag value from previous command|

    Your file should look similar to the example below:

    ```yaml
    controller:
      appprotect: 
        enable: true
      appprotectdos:
        enable: true
      enableSnippets: true
      image:
        repository: ghcr.io/codygreen/nginx-plus-ingress
        tag: 2.3.0-SNAPSHOT-a88b7fe
      nginxPlus: true
      nginxStatus:
        allowCidrs: 9000
        port: 9000
      serviceAccount:
        imagePullSecretName: ghcr
    prometheus:
      create: true
    ```

1. Save the file. Next, you will need to update the NGINX Plus Ingress Argo CD manifest to match your environment.  

1. Open the `manifests/nginx-ingress-subchart.yaml` file in your forked version of the **infra** repository.

1. Find the following variables and replace them with your information:

    | Variable        | Value           |
    |-----------------|-----------------|
    | \<GITHUB_USER\> | github username |

    Your file should look similar to the example below:

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: nginx-plus-ingress
      namespace: argocd
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
      source:
        path: Infra/charts/nginx-plus-ingress
        repoURL: https://github.com/oteizaa/OCP-NIC-XC.git
        targetRevision: HEAD
      destination:
        namespace: nginx-ingress
        server: https://kubernetes.default.svc
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
          - CreateNamespace=true
          - ApplyOutOfSyncOnly=true
    ```

1. Save the file.

1. Stage both changed files, and commit them to your local **infra** repository.

1. Push the changes to your remote **infra** repository.

## Install NGINX Plus Ingress Argo CD Application

1. Now that we have the base requirements ready, we can add the NGINX Plus Ingress application to Argo CD with the following command:

    ```bash
    kubectl apply -f Infra/manifests/nginx-ingress-subchart.yaml
    ```

## Verify Install

Now that NGINX Plus Ingress Controller has been installed, we need to check that our pods are up and running.

## Verify Deployment

1. To check our pod run the following command:

    ```bash
    kubectl get pods -n nginx-ingress
    ```

    The output should look similar to:

    ```shell
    NAME                                                READY   STATUS    RESTARTS   AGE
    nginx-plus-ingress-nginx-ingress-7547565fbc-f8nqj   1/1     Running   0          55m
    ```

1. To check our service run the following command:

    ```bash
    kubectl get svc -n nginx-ingress
    ```

    The output should look similar to:

    ```shell
    NAME                               TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
    nginx-plus-ingress-nginx-ingress   LoadBalancer   10.43.129.144   10.1.1.5      80:31901/TCP,443:31793/TCP   57m
    ```

## Inspect Pod Details

1. Now that we know our NGINX Ingress Controller Pod is up and running, let's dig into some of the pod details.

    ```bash
    NIC_POD=`kubectl get pods -n nginx-ingress -o json | jq '.items[0].metadata.name' -r`
    kubectl describe pod $NIC_POD -n nginx-ingress
    ```

    The output should look similar to:

    ```bash
    Name:         nginx-plus-ingress-nginx-ingress-785b67bf4-vgtdl
    Namespace:    nginx-ingress
    Priority:     0
    Node:         k3s/10.1.1.5
    Start Time:   Wed, 06 Jul 2022 09:07:17 -0700
    Labels:       app=nginx-plus-ingress-nginx-ingress
                  pod-template-hash=785b67bf4
    Annotations:  prometheus.io/port: 9113
                  prometheus.io/scheme: http
                  prometheus.io/scrape: true
    Status:       Running
    IP:           10.42.0.22
    IPs:
      IP:           10.42.0.22
    Controlled By:  ReplicaSet/nginx-plus-ingress-nginx-ingress-785b67bf4
    Containers:
      nginx-plus-ingress-nginx-ingress:
        Container ID:  containerd://69e9e416438c2cc2330df627cc7605640f6c196092a4ea3f7ff421c3bcfbbcd7
        Image:         ghcr.io/codygreen/nginx-plus-ingress:2.3.0-SNAPSHOT-a88b7fe
        Image ID:      ghcr.io/codygreen/nginx-plus-ingress@sha256:6b480db30059249d90d4f2d9d8bc2012af8c76e9b25799537f4b7e5a4a2946ca
        Ports:         80/TCP, 443/TCP, 9113/TCP, 8081/TCP
        Host Ports:    0/TCP, 0/TCP, 0/TCP, 0/TCP
        Args:
          -nginx-plus=true
          -nginx-reload-timeout=60000
          -enable-app-protect=true
          -enable-app-protect-dos=true
          -app-protect-dos-debug=false
          -app-protect-dos-max-daemons=0
          -app-protect-dos-max-workers=0
          -app-protect-dos-memory=0
          -nginx-configmaps=$(POD_NAMESPACE)/nginx-plus-ingress-nginx-ingress
          -default-server-tls-secret=$(POD_NAMESPACE)/nginx-plus-ingress-nginx-ingress-default-server-tls
          -ingress-class=nginx
          -health-status=false
          -health-status-uri=/nginx-health
          -nginx-debug=false
          -v=1
          -nginx-status=true
          -nginx-status-port=9000
          -nginx-status-allow-cidrs=0.0.0.0/0
          -report-ingress-status
          -external-service=nginx-plus-ingress-nginx-ingress
          -enable-leader-election=true
          -leader-election-lock-name=nginx-plus-ingress-nginx-ingress-leader-election
          -enable-prometheus-metrics=true
          -prometheus-metrics-listen-port=9113
          -prometheus-tls-secret=
          -enable-custom-resources=true
          -enable-snippets=true
          -enable-tls-passthrough=false
          -enable-preview-policies=false
          -enable-cert-manager=false
          -enable-oidc=false
          -ready-status=true
          -ready-status-port=8081
          -enable-latency-metrics=false
        State:          Running
          Started:      Wed, 06 Jul 2022 09:07:24 -0700
        Ready:          True
        Restart Count:  0
        Readiness:      http-get http://:readiness-port/nginx-ready delay=0s timeout=1s period=1s #success=1 #failure=3
        Environment:
          POD_NAMESPACE:  nginx-ingress (v1:metadata.namespace)
          POD_NAME:       nginx-plus-ingress-nginx-ingress-785b67bf4-vgtdl (v1:metadata.name)
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-tp2v4 (ro)
    Conditions:
      Type              Status
      Initialized       True
      Ready             True
      ContainersReady   True
      PodScheduled      True
    Volumes:
      kube-api-access-tp2v4:
        Type:                    Projected (a volume that contains injected data from multiple sources)
        TokenExpirationSeconds:  3607
        ConfigMapName:           kube-root-ca.crt
        ConfigMapOptional:       <nil>
        DownwardAPI:             true
    QoS Class:                   BestEffort
    Node-Selectors:              <none>
    Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:                      <none>
    ```

    Some items of interest from the output:

    - Ports:
      - 80/443: http/https traffic
      - 8081: readiness
      - 9113: Prometheus


# Install Prometheus via Argo CD

In this step, you will use GitOps to install Prometheus leveraging Argo CD.

## Update Argo CD Application Manifest

You will need to update the Prometheus Argo CD manifest to match your environment.

1. Open the `manifests/prometheus-subchart.yaml` file in your forked version of the **infra** repository.

1. Find the following variables and replace them with your information:

    | Variable        | Value           |
    |-----------------|-----------------|
    | \<GITHUB_USER\>   | github username |

    Your file should look similar to the example below:

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: prometheus
      namespace: argocd
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
      source:
        path: charts/prometheus
        repoURL: https://github.com/codygreen/modern_app_jumpstart_workshop.git
        targetRevision: HEAD
      destination:
        namespace: monitoring
        server: https://kubernetes.default.svc
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
          - CreateNamespace=true
          - ApplyOutOfSyncOnly=true
    ```

1. Save the file.

1. Stage the changes, and commit to your local **infra** repository.

1. Push the changes to your remote **infra** repository.

## Deploy the manifest

1. To deploy the Prometheus Argo CD application, run the following command:

    ```bash
    kubectl apply -f manifests/prometheus-subchart.yaml
    ```

## Verify Install

You should now see a new Prometheus application in your Argo CD dashboard. Click on the Prometheus application and verify there are no errors.

# Install Grafana via Argo CD

In this step, you will use GitOps to install Grafana leveraging Argo CD.

## Update Argo CD Application Manifest

You will need to update the Grafana Argo CD manifest to match your environment.  

1. Open the `manifests/grafana-subchart.yaml` file in your forked version of the **infra** repository.

1. Find the following variables and replace them with your information:

    | Variable        | Value           |
    |-----------------|-----------------|
    | \<GITHUB_USER\>   | github username |

    Your file should look similar to the example below:

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: grafana
      namespace: argocd
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
      source:
        path: charts/grafana
        repoURL: https://github.com/codygreen/modern_app_jumpstart_workshop.git
        targetRevision: HEAD
      destination:
        namespace: monitoring
        server: https://kubernetes.default.svc
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
          - CreateNamespace=true
          - ApplyOutOfSyncOnly=true
    ```

1. Save the file.

1. Stage the changes, and commit to your local **infra** repository.

1. Push the changes to your remote **infra** repository.

## Deploy the manifest

1. To deploy the Grafana Argo CD application, run the following command:

    ```bash
    kubectl apply -f manifests/grafana-subchart.yaml
    ```

1. You should now see a new Grafana application in your Argo CD dashboard. Click on the Grafana application and verify there are no errors.

    > **Note:** A Prometheus datasource and a Dashboard for NGINX Ingress Controller have been pre-configured in Grafana. These can be seen in your `charts/grafana/values.yaml` file. We will utilize these in an upcoming exercise.


# Install Brewz with ArgoCD

In this section, you will deploy the Brewz microservices using Argo CD.

## Update Argo CD Application Manifest

You will need to update the Brewz Argo CD manifest to match your environment.  

1. Open the `manifests/brewz-subchart.yaml` file in your forked version of the **primary** repository.

1. Find the following variables and replace them with your information:

    | Variable        | Value           |
    |-----------------|-----------------|
    | \<GITHUB_USER\>   | github username |

    Your file should look similar to the example below:

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: brewz
      namespace: argocd
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
      source:
        path: manifests/brewz
        repoURL: https://github.com/codygreen/modern_app_jumpstart_workshop.git
        targetRevision: HEAD
      destination:
        namespace: default
        server: https://kubernetes.default.svc
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
    ```

1. Save the file.

1. Stage the changes, and commit to your local **primary** repository.

1. Push the changes to your remote **primary** repository.

## Deploy the manifest

1. To deploy the Brewz Argo CD application, run the following command:

    ```bash
    kubectl apply -f manifests/brewz-subchart.yaml
    ```

## View the Argo CD Application

1. Open the Argo CD UDF Access Method under the K3s server
  ![Argo CD Sync](../assets/argo_sync.jpg)

1. Click on the argo-cd-demo application in the Argo CD UI and inspect the deployed services and policies.

    > **Note:** You should see the individual services as well as the `rate-limit-policy` resource from day 1 of the workshop.

## Inspect the NGINX Ingress Controller Configuration

Now that Argo CD has deployed out application let's take a look at the NGINX Ingress Controller Virtual Server resources.

**Note:** Refer to the [VirtualServer docs](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/) for more information.

1. Run the following on your local machine:

    ```bash
    kubectl get virtualserver
    ```

    Your output should look similar to:

    ```bash
    NAME    STATE   HOST               IP         PORTS      AGE
    brewz   Valid   brewz.f5demo.com   10.1.1.5   [80,443]   29m
    ```

1. Let's take a deeper look at the configuration:

    ```shell
    kubectl describe virtualserver brewz
    ```

    Your output should look similar to:

    ```shell
    Name:         brewz
    Namespace:    default
    Labels:       app.kubernetes.io/instance=brewz
    Annotations:  <none>
    API Version:  k8s.nginx.org/v1
    Kind:         VirtualServer
    Metadata:
      Creation Timestamp:  2022-07-06T16:51:10Z
      Generation:          1
      Managed Fields:
        API Version:  k8s.nginx.org/v1
        Fields Type:  FieldsV1
        fieldsV1:
          f:metadata:
            f:annotations:
              .:
              f:kubectl.kubernetes.io/last-applied-configuration:
            f:labels:
              .:
              f:app.kubernetes.io/instance:
          f:spec:
            .:
            f:host:
            f:http-snippets:
            f:routes:
            f:server-snippets:
            f:upstreams:
        Manager:      argocd-application-controller
        Operation:    Update
        Time:         2022-07-06T16:51:10Z
        API Version:  k8s.nginx.org/v1
        Fields Type:  FieldsV1
        fieldsV1:
          f:status:
            .:
            f:externalEndpoints:
            f:message:
            f:reason:
            f:state:
        Manager:         Go-http-client
        Operation:       Update
        Subresource:     status
        Time:            2022-07-06T16:51:11Z
      Resource Version:  6304
      UID:               c137a6c1-7072-4afb-a12d-16b727936b13
    Spec:
      Host:             brewz.f5demo.com
      Routes:
        Action:
          Pass:  spa
        Matches:
          Action:
            Pass:  spa-dark
          Conditions:
            Cookie:  app_version
            Value:   dark
        Path:        /
        Action:
          Pass:  api
        Path:    /api
        Policies:
          Name:  rate-limit-policy
        Action:
          Proxy:
            Rewrite Path:  /api/inventory
            Upstream:      inventory
        Path:              /api/inventory
        Action:
          Proxy:
            Rewrite Path:  /api/recommendations
            Upstream:      recommendations
        Path:              /api/recommendations
        Action:
          Proxy:
            Rewrite Path:  /images
            Upstream:      api
        Path:              /images
        Upstreams:
        Name:     spa
        Port:     80
        Service:  spa
        Name:     spa-dark
        Port:     80
        Service:  spa-dark
        Name:     api
        Port:     8000
        Service:  api
        Name:     inventory
        Port:     8002
        Service:  inventory
        Name:     recommendations
        Port:     8001
        Service:  recommendations
    Status:
      External Endpoints:
        Ip:     10.1.1.5
        Ports:  [80,443]
      Message:  Configuration for default/brewz was added or updated
      Reason:   AddedOrUpdated
      State:    Valid
    Events:     <none>
    ```

    A few items of interest in the output:

    - Spec.Routes.Action:
      - "/" -> spa
      - "/" with cookie `app_version=dark` -> spa-dark
      - "/api" -> api
        - with a policy named rate-limit-policy
      - "/images" -> rewrite policy to the api upstream
      - "/api/inventory" -> /api/inventory of inventory service
      - "/api/recommendations" -> /api/recommendations of recommendations service


# VirtualServer and VirtualServerRoute Resources

VirtualServer and VirtualServerRoute resources were added into NGINX Ingress Controller started in version 1.5 and are implemented via [Custom Resources (CRDs)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/).

The resources enable use cases not supported with the Ingress resource, such as traffic splitting and advanced content-based routing.

More detailed information can be found in the [VirtualServer and VirtualServerRoute Resources docs](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources).

Let's take a look at common resources for a modern application.

> **Note:** You will use your forked version of the **primary** repository for this portion of the lab.

## Action

The action resource defines an action to perform for a request and is the basis for our Brewz demo application.

### Pass

The *pass* action passes the request to an upstream that is defined in the resource.

In the Brewz `virtual-server.yaml` manifest, the *spa* and *api* services leverage this method.

```yaml
...
  upstreams:
    - name: spa
      service: spa
      port: 80
    - name: api
      service: api
      port: 8000
...
  routes:
    - path: /
      action:
        pass: spa
    - path: /api
      action:
        pass: api
...
```

### Redirect

The *redirect* action redirects a request to a provided URL.

### Return

The *return* action returns a preconfigured response.

### Proxy

The *proxy* action passes a request to an upstream with the ability to modify the request/response.

In the Brewz `virtual-server.yaml` manifest, the */images* path uses this method to proxy requests to the api service's */images* path.

```yaml
...
    - path: /images
      action:
        proxy:
          upstream: api
          rewritePath: /images
...
```

## Upstreams

The upstream defines a destination for the routing configuration. The upstream's name must be a valid DNS label as defined in RFC 1035.

In the Brewz `virtual-server.yaml` manifest, we define a very simple upstream configuration for the *spa* and *api* services:

```yaml
...
  upstreams:
    - name: spa
      service: spa
      port: 80
    - name: api
      service: api
      port: 8000
...
```

While this configuration meets our requirements for a lab, in a production environment you may need more advanced configurations.  For example, if we knew that the API server could only handle 32 concurrent connections then we may want to modify the manifest to include the *max-conns* attribute:

```yaml
...
  upstreams:
    - name: spa
      service: spa
      port: 80
    - name: api
      service: api
      port: 8000
      max-conns: 32
...
```

For a full list of Upstream attributes, please refer to the [docs](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/#upstream).

### HealthCheck

One of the advantages the NGINX Plus Ingress Controller provides is the ability to perform health checks on your upstreams. This can be very useful in situations like the Brewz API which is dependent on a MongoDB database to function correctly.  By checking the APIs' custom /stats API, we can determine if the API server is functioning correctly.

1. In VSCode, open the `manifests/brewz/virtual-server.yaml` file and add a `healthCheck` resource to the `api` upstream; example below.

    ```yaml
    ---
    apiVersion: k8s.nginx.org/v1
    kind: VirtualServer
    metadata:
      name: brewz
    spec:
      host: brewz.f5demo.com
      upstreams:
        - name: spa
          service: spa
          port: 80
        - name: api
          service: api
          port: 8000
          healthCheck:
            enable: true
            path: /api/stats
            interval: 20s
            jitter: 3s
            port: 8000
        - name: inventory
          service: inventory
          port: 8002
        - name: recommendations
          service: recommendations
          port: 8001
        - name: spa-dark
          service: spa-dark
          port: 80
      routes:
        - path: /
          matches:
            - conditions:
              - cookie: "app_version"
                value: "dark"
              action:
                pass: spa-dark
          action:
            pass: spa
        - path: /api
          policies:
            - name: rate-limit-policy
          action:
            pass: api
        - path: /api/inventory
          action:
            proxy:
              upstream: inventory
              rewritePath: /api/inventory
        - path: /api/recommendations
          action:
            proxy:
              upstream: recommendations
              rewritePath: /api/recommendations
        - path: /images
          action:
            proxy:
              upstream: api
              rewritePath: /images
    ```

1. Commit the `manifests/brewz/virtual-server.yaml` file to your local repository, then push it to your remote repository. Argo CD will pick up the most recent changes, and deploy them for you.

1. Run the following command on the K3s server via the UDF *SSH* or *Web Shell* Access Methods to test that our API services is still up and has a health check:

    ```bash
    # Find the Brewz Access Method's Host
    HOST=`curl -s metadata.udf/deployment | jq '.deployment.components[] | select(.name == "k3s") | .accessMethods.https[] | select(.label == "Brewz") | .host' -r`
    curl -k https://$HOST/api/products/123
    ```

## ErrorPage

While the Brews developers were able to break their monolith application into microservices, their APIs are not always returning a JSON response.  A good example is when you lookup a product that does not exist.  The API returns a 400 HTTP response code but the body payload is *"Could not find the product!"*.

1. Run the following command on the K3s server via the UDF *SSH* or *Web Shell* Access Methods to test this output:

    ```bash
    # Find the Brewz Access Method's Host
    HOST=`curl -s metadata.udf/deployment | jq '.deployment.components[] | select(.name == "k3s") | .accessMethods.https[] | select(.label == "Brewz") | .host' -r`
    curl -k https://$HOST/api/products/1234
    ```

    > Ideally, the development team will fix this issue in the API code but we can also help by performing a quick fix via our VirtualServer configuration.

1. In VSCode, open the `manifests/brewz/virtual-server.yaml` file and add an `errorPages` resource to the `routes` -> `api` path; example below.

    ```yaml
    ---
    apiVersion: k8s.nginx.org/v1
    kind: VirtualServer
    metadata:
      name: brewz
    spec:
      host: brewz.f5demo.com
      upstreams:
        - name: spa
          service: spa
          port: 80
        - name: api
          service: api
          port: 8000
          healthCheck:
            enable: true
            path: /api/stats
            interval: 20s
            jitter: 3s
            port: 8000
      routes:
        - path: /
          action:
            pass: spa
        - path: /api
          policies:
            - name: rate-limit-policy
          action:
            pass: api
          errorPages:
            - codes: [404]
              return:
                code: 404
                type: application/json
                body: |
                  {\"msg\": \"Could not find the product!\"}
                headers:
                  - name: x-debug-original-status
                    value: ${upstream_status}
        - path: /images
          action:
            proxy:
              upstream: api
              rewritePath: /images
    ```

1. Commit the `manifests/brewz/virtual-server.yaml` file to your local repository, then push it to your remote repository. Argo CD will pick up the most recent changes, and deploy them for you.

1. Now, check that an unknown product returns a JSON object by running the following command on the K3s server:

    ```bash
    # Find the Brewz Access Method's Host
    HOST=`curl -s metadata.udf/deployment | jq '.deployment.components[] | select(.name == "k3s") | .accessMethods.https[] | select(.label == "Brewz") | .host' -r`
    curl -k https://$HOST/api/products/1234
    ```

    > Your output should look like: `{"msg": "Could not find the product!"}`

## TLS

A common requirement for most websites today is to leverage encryption to secure the communication between the client and the server. While this would be a critical requirement for an e-commerce site like our Brewz demo app, many enterprise customers also require encryption due to search engines, such as Google, demoting their rank in search results if the website does not offer encryption.

In this step, we will add TLS encryption to the Brewz VirtualServer resource.

### Create a certificate

Since you are running this lab in a closed ecosystem (UDF), you do not have the ability to easily create external DNS records and obtain a public TLS certificate; we will look at this ability in another lab leveraging F5 Distributed Cloud. Due to this limitation, we will create a self-signed certificate.

1. Run the following commands via SSH on the K3s server using the *SSH* or *Web Shell* UDF Access Methods:

    ```bash
    # Create the key and cert
    sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/brewz-selfsigned.key -out /etc/ssl/certs/brewz-selfsigned.crt
    ```

    OpenSSL will prompt you with questions about your cert request:

    - Country Name: US
    - State or Province Name: WA
    - Locality Name: Seattle
    - Organization Name: F5
    - Organizational Unit Name: Brewz
    - Common Name: brewz.f5demo.com
    - Email Address: brewsz@f5demo.com

### Create K8s Secret for the Cert and Key

Now that we have a self-signed certificate, we need to add it to our K8s cluster as a secret.

1. Run the following commands via SSH on the K3s server using the *SSH* or *Web Shell* UDF Access Methods:

    ```bash
    sudo kubectl create secret tls brewz-tls --key=/etc/ssl/private/brewz-selfsigned.key --cert=/etc/ssl/certs/brewz-selfsigned.crt
    ```

1. Now that your secret is created, let's take a look at it. Run the following command from your laptop:

    ```bash
    kubectl describe secret brewz-tls
    ```

    Your output should look similar to:

    ```shell
    Name:         brewz-tls
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>

    Type:  kubernetes.io/tls

    Data
    ====
    tls.crt:  1164 bytes
    tls.key:  1704 bytes
    ```

### Modify VirtualServer Resource

The final step is to update our Brewz VirtualServer resource to leverage the new TLS certificate.

1. In VSCode, open the `manifests/brewz/virtual-server.yaml` file and add the following fields to the virtual server:

    ```yaml
    tls:
      secret: brewz-tls
    ```

    The final file should look like the example below:

    ```yaml
    ---
    apiVersion: k8s.nginx.org/v1
    kind: VirtualServer
    metadata:
      name: brewz
    spec:
      host: brewz.f5demo.com
      tls:
        secret: brewz-tls
      upstreams:
        - name: spa
          service: spa
          port: 80
        - name: api
          service: api
          port: 8000
          healthCheck:
            enable: true
            path: /api/stats
            interval: 20s
            jitter: 3s
            port: 8000
        - name: inventory
          service: inventory
          port: 8002
        - name: recommendations
          service: recommendations
          port: 8001
        - name: spa-dark
          service: spa-dark
          port: 80
      routes:
        - path: /
          matches:
            - conditions:
              - cookie: "app_version"
                value: "dark"
              action:
                pass: spa-dark
          action:
            pass: spa
        - path: /api
          policies:
            - name: rate-limit-policy
          action:
            pass: api
          errorPages:
            - codes: [404]
              return:
                code: 404
                type: application/json
                body: |
                  {\"msg\": \"Could not find the product!\"}
                headers:
                  - name: x-debug-original-status
                    value: ${upstream_status}
        - path: /api/inventory
          action:
            proxy:
              upstream: inventory
              rewritePath: /api/inventory
        - path: /api/recommendations
          action:
            proxy:
              upstream: recommendations
              rewritePath: /api/recommendations
        - path: /images
          action:
            proxy:
              upstream: api
              rewritePath: /images

    ```

1. Commit the `manifests/brewz/virtual-server.yaml` file to your local repository, then push it to your remote repository. Argo CD will pick up the most recent changes, and deploy them for you.

### Check the status of our virtual server

1. Check the state of the Virtual Server, it should be *Valid*:

    ```bash
    kubectl get vs
    ```

1. Open a **WebShell** for the K3s server in the UDF deployment

1. Check the SSL certificate on the NGINX Ingress, notice the CN is NGINXIngressController

    ```bash
    echo | openssl s_client -connect 10.1.1.5:443 2> /dev/null | grep subject=
    ```

1. Now, check the SSL certificate for the **brewz.f5demo.com** Virtual Server, notice the certificate information you entered when you created the cert:

    ```bash
    echo | openssl s_client -connect 10.1.1.5:443  -servername brewz.f5demo.com 2> /dev/null | grep subject=
    ```

## Next Steps

# Deployment Pattern Example: Canary

 The development team has been hard at work updating the Brewz application to expand its feature set. One of the changes requested is to the recommendation service. They want to change the algorithm that is used when suggesting additional products that the shopper can buy. They would ideally like to deploy this with no disruption in production. In this exercise, we will examine metrics that can be used to determine whether there are errors in the upstream services, as we roll out the new version of the recommendation service using the Canary deployment pattern.

## Examine NGINX Dashboard in Grafana

1. Click on the **Grafana** access method in the **k3s** component in the UDF deployment.

1. When presented for login credentials, enter `admin` as the username. To acquire the password, you must run the following command from your local machine to interrogate the K8s API for the secret containing the password:

    ```bash
    kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
    ```

1. Select the **NGINX Plus Ingress Controller** dashboard from the Grafana **Dashboards** menu. The dashboard should appear:

    <img src="../assets/grafana-nginx-ingress-dashboard.png" alt="NGINX Plus Ingress Controller dashboard" width="600"/>

1. By default, this dashboard does not automatically refresh. Click the refresh icon in the upper right of the screen and set the refresh value to 10 seconds.

    <img src="../assets/grafana-refresh.png" alt="Grafana dashboard refresh" width="160"/>

    > Note that there are various filters you can use to examine a subset of the data. By default, the dashboard will consume NGINX Plus Prometheus metrics from all namespaces, so you may wish to filter on the `default` namespace. Additionally, you may also filter data that appears in the **Upstream Metrics** portion of the dashboard to specific upstreams by use of the **Upstream Server** menu.

1. Open the **Brewz** UDF access method of the **k3s** component, and exercise various functions of the application.

1. Return to the Grafana dashboard. You should start to see **Ingress Metrics** such as **Success Rates Over Time** and **Upstream Success Rate** charts start to trend upward. This is a good indication that the application is operating correctly.

## Deploy the Recommendations v2 Service

The development team has developed and created a container image of the recommendations service in the registry. We will deploy this new version of the service alongside the existing version, but at first only send a portion of incoming requests to this new version of the service. We will monitor the health of this new service's upstream in the Grafana dashboard, and will either continue the rollout, or back out the change if errors occur. Let's do it.

> **Note:** You will use your forked version of the **primary** repository for this portion of the lab.

1. In VSCode, append the following new `Deployment` and `Service` resources to the `manifests/brewz/app.yaml` file, and save it:

    ```yaml

    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: recommendations-v2
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: recommendations-v2
      template:
        metadata:
          labels:
            app: recommendations-v2
        spec:
          containers:
            - name: recommendations-v2
              image: ghcr.io/f5devcentral/spa-demo-app-recommendations:sha-8123c6f
              ports:
                - containerPort: 8001
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: recommendations-v2
    spec:
      ports:
        - port: 8001
          targetPort: 8001
          protocol: TCP
          name: http
      selector:
        app: recommendations-v2

    ```

1. Append the following yaml snippet to the list of `upstreams` in the `manifests/brewz/virtual-server.yaml` file:

    ```yaml
        - name: recommendations-v2
          service: recommendations-v2
          port: 8001
    ```

1. Modify the existing `/api/recommendations` path in the `routes` section of the file so it looks like this and save it - ensure you remove the existing action:

    ```yaml
        - path: /api/recommendations
          splits:
            - weight: 90
              action:
                proxy:
                  upstream: recommendations
                  rewritePath: /api/recommendations
            - weight: 10
              action:
                proxy:
                  upstream: recommendations-v2
                  rewritePath: /api/recommendations
    ```

    > **Note:** The result of these changes to the file will configure NGINX Ingress Controller to route roughly 90% of requests to the `/api/recommendations` path to the `recommendations` upstream, and the remaining 10% to the `recommendations-v2` upstream.

1. Commit the `manifests/brewz/virtual-server.yaml` and `manifests/brewz/app.yaml` files to your local repository, then push them to your remote repository. Argo CD will pick up the most recent changes, and deploy them for you.

    > **Note:** Once the configuration is deployed, NGINX Ingress Controller will reload NGINX, and the **Reloads** metric on the Grafana dashboard should increment.

1. Once deployed, use the **Hey** utility on your laptop to request the **recommendations** service directly as if the Brews SPA application was doing so:

    ```bash
    BREWZ_URL=<Your Brewz UDF access method url>
    hey -n 2000 -c 4 -q 10 $BREWZ_URL/api/recommendations
    ```

1. As this is running, monitor the success and error rates in the Grafana dashboard. What do you see? It looks like an upstream is reporting server errors. The **recommendations-v2** service (in association to its upstream) is reporting the errors. Also, when the **Hey** utility completes running, you should notice roughly 10% of the requests returned 500 errors in its summary.

    <img src="../assets/grafana-nginx-ingress-dashboard-errors.png" alt="NGINX Plus Ingress Controller dashboard errors" width="600"/>

## Rollback the Deployment

The DevOps and the application owners aren't willing to allow this error condition to continue, so it is decided to route all traffic back to the older version of the recommendations service so that the team can investigate offline without affecting customers. While Kubernetes and Argo CD both have rollback capabilities, we have made the decision to effectively revert the change so that all traffic is once again routed to the older version of the recommendations service.

1. In VSCode, open a terminal and run the following commands to revert your previous git commit:

    ```bash
    # Find your last commit ID
    git log -1

    # revert your commit
    git revert your_commit_id

    # Update origin 
    git push origin
    ```

1. Argo CD will pick up the most recent changes, and deploy them for you. Check the brewz `VirtualServer` resource under the `brewz` application to check that the revert was successful.

1. Once the revert is successful, use the **Hey** utility on your laptop to request the **recommendations** service directly as if the Brews SPA application was doing so:

    ```bash
    BREWZ_URL=<Your Brewz UDF access method url>
    hey -n 2000 -c 4 -q 10 $BREWZ_URL/api/recommendations
    ```

1. As this is running, monitor the success and error rates in the Grafana dashboard. What do you see? The upstream error rates should return to zero and no longer reporting server errors.

## Next Steps

Next, you will [add API protection with NGINX App Protect WAF](waf.md).


# Adding API Protection with NGINX App Protect WAF

The SecOps engineers monitoring the Brewz application have discovered that there appear to be nefarious users and/or bots that have been attempting to perform injection attacks against the API. Additionally, these actors have been spotted trying to exploit destructive HTTP verbs against the API, searching for ways to mine data out of their APIs as well as alter/destroy data.

NGINX Ingress Controller has the ability to configure the NGINX App Protect WAF which will be used to defend against these kind of attacks. The standard "out of the box" capability of NAP WAF will protect against commonly observed injection attacks. However, how can we configure it with knowledge of our Brewz APIs in order to exclude illegitimate API traffic? NAP WAF can build a complex "positive security" policy using an [OpenAPI spec](https://spec.openapis.org/oas/latest.html), which the Brews developers have already produced to describe their APIs. It is just a matter of presenting The OpenAPI spec and NAP policy to NGINX Ingress Controller expressed as custom resources, and it will start enforcing.

## Abuse the API

1. In your local shell, set the `BREWZ_URL` variable with the Brewz UDF Access method in the k3s component (without the trailing slash, and no path parameters). Example:

    ```bash
    BREWZ_URL=https://0d13e993-07ee-4433-91b4-a91788f78847.access.udf.f5.com
    ```

1. Use cURL to make requests against the API. This first request violates the implicitly expected type of `userId` in the user API by injecting alphanumeric characters into it:

    ```bash
    curl -k -X GET "$BREWZ_URL/api/users/12345b/cart"
    ```

    You should receive a response of `"Could not find user!"`, as `12345b` in not a valid user id format.

1. The next request will attempt a `POST` without the expected payload, and with an invalid and unexpected format of the `userId` parameter:

    ```bash
    curl -k -X POST "$BREWZ_URL/api/users/12345b/cart"
    ```

    > The request should not return anything and just time out. Why?

1. Open a new terminal. Tail the logs in the API pod to see what is going on. You will likely need to set the `KUBECONFIG` variable in this new terminal, so we include this command here:

    ```bash
    export KUBECONFIG=~/Downloads/config-udf.yaml
    API_POD=`kubectl get pods -o json | jq '.items[] | select(.metadata.name | startswith("api-")) | .metadata.name' -r`
    kubectl logs $API_POD -f
    ```

    > You may notice that there is an unhandled exception being logged, causing the request to timeout. This is obviously something that should be addressed in code, but we may be able to do something about it in the mean time.

## Create and Deploy Security Policy

We will deploy the NAP WAF policy that is referencing the OpenAPI spec that the Brewz developers provided us. This should stop unexpected and invalid requests from making it through NGINX Ingress Controller, performing as an API Gateway.

> **Note:** You will use your forked version of the **primary** repository for this portion of the lab.

1. Copy `waf-ap-logconf.yaml`, `waf-ap-policy.yaml` and `waf-policy.yaml` from your `docs/ingress/source-manifests` folder into your `manifests/brewz` folder.

1. Update your `manifests/brewz/virtual-server.yaml` file to add the `waf-policy` policy reference to the `VirtualServer` spec as in this snippet:

    ```yaml
    ---
    apiVersion: k8s.nginx.org/v1
    kind: VirtualServer
    metadata:
      name: brewz
    spec:
      host: brewz.f5demo.com
      tls:
        secret: brewz-tls
      policies:
        - name: waf-policy
      upstreams:
        - name: spa
          service: spa
          port: 80

    ...
    ```

1. Commit the copied and modified files to your local repository, then push them to your remote repository. Argo CD will pick up the most recent changes, and deploy them for you.

1. While it is deploying, review the files you copied:

    - `waf-ap-policy.yaml` is the NAP policy itself, packaged into an `APPolicy` custom resource type. It is set to global blocking, and enables blocking for specific violations that we would like to have enforced for the Brewz APIs. Note that the OpenAPI file itself is referenced at the bottom of the policy file. Once the policy is loaded into the ingress controller and presented to NAP, it will be downloaded from the referenced [public GitHub URL](https://raw.githubusercontent.com/f5devcentral/modern_app_jumpstart_workshop/main/docs/ingress/source-manifests/oas.yaml). You are free to examine this file now, or later in the exercise.
    - `waf-ap-logconf.yaml` is the logging configuration that NAP WAF will use, packaged as an `APLogConf` custom resource. Note that it is set to log `blocked` requests only.
    - `waf-policy.yaml` is a `Policy` custom resource that stitches together the `APPolicy` and `APLogConf` resources. This is the resource that we referenced and attached to the `VirtualServer` resource above.

## Monitor NAP WAF Security Events

If you examine the contents of the `APLogConf` resource contained in `manifests/brewz/waf-ap-logconf.yaml` file, you will notice that we have configured NAP WAF to log to `stderr` rather than to a file destination. NGINX Ingress Controller is already logging both access log entries and configuration events to the `stdout` and `stderr` log stream and are viewable with `kubectl logs` executed on its pod. NAP WAF violation logs will now appear in this log stream as well.

1. Open a new terminal and tail ("follow" with the `-f` option) the NIC pod logs and stream them to your terminal. You will likely need to set the `KUBECONFIG` variable in this new terminal, so we include this command here:

    ```bash
    export KUBECONFIG=~/Downloads/config-udf.yaml
    NIC_POD=`kubectl get pods -n nginx-ingress -o json | jq '.items[0].metadata.name' -r`
    kubectl logs $NIC_POD -n nginx-ingress -f
    ```

    We will use this log stream in the next section.

    > **Note:** At times, the log stream may stop. If you are not seeing events appear after some time, type `ctrl+c` and attempt to stream logs again.

## Test for Efficacy

1. We are going to attempt the requests we attempted before:

    ```bash
    curl -k -X GET "$BREWZ_URL/api/users/12345b/cart"

    curl -k -X POST "$BREWZ_URL/api/users/12345b/cart"

    ```

    Both of these requests should return a response similar to:

    ```json
    {"supportID": "387465341565226259"}
    ```

1. Examine the log stream. Notice there are NAP WAF log entries that contain violations of `VIOL_URL_CONTENT_TYPE` and `VIOL_PARAMETER_DATA_TYPE` in the logs. This is because NAP WAF is now enforcing expectations associated with requests as dictated by the OpenAPI spec we provided NAP WAF.

1. Attempt a new request that uses a properly formatted `userId`, but does not include the correct content type:

    ```bash
    curl -k -X POST "$BREWZ_URL/api/users/12345/cart"
    ```

    In the log stream, notice a violation of `VIOL_URL_CONTENT_TYPE` appears. This is due to the fact that line 117 in the `oas.yaml` spec file stipulates that requests to this http URI and verb must be of content type `application/json`.

1. Attempt a new request that includes the expected content type, yet violates the request payload expectations:

    ```bash
    curl -k -d '{"productId":"1234r"}' -H "Content-Type: application/json" -X POST $BREWZ_URL/api/users/12345/cart
    ```

    In the log stream, notice a violation of `VIOL_JSON_SCHEMA` appears. This is due to the fact that line 120 in the `oas.yaml` spec file stipulates that requests must include a `productId` property, that is a string that is coercable into a number with a minimum of 3 digits.

1. Finally, send a valid request and note that it is successful and returns all the products in the user cart:

    ```bash
    curl -k -d '{"productId":"123"}' -H "Content-Type: application/json" -X POST $BREWZ_URL/api/users/12345/cart
    ```

## End of Lab

