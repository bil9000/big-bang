# Big Bang Quick Start

[[_TOC_]]

## Overview

This quick start guide explains in beginner-friendly terminology how to complete the following tasks in under an hour:

1. Turn a virtual machine (VM) into a k3d single-node Kubernetes cluster.
1. Deploy Big Bang on the cluster using a demonstration and local development-friendly workflow.

    > Note: This guide mainly focuses on the scenario of deploying Big Bang to a remote VM with enough resources to run Big Bang [(see step 1 for recommended resources)](#step-1:-provision-a-virtual-machine). If your workstation has sufficient resources, or you are willing to disable packages to lower the resource requirements, then local development is possible. This quick start guide is valid for both remote and local deployment scenarios.

1. Customize the demonstration deployment of Big Bang.

## Important Background Contextual Information

`BLUF:` This quick start guide optimizes the speed at which a demonstrable and tinker-able deployment of Big Bang can be achieved by minimizing prerequisite dependencies and substituting them with quickly implementable alternatives. Refer to the [Customer Template Repo](https://repo1.dso.mil/platform-one/big-bang/customers/template) for guidance on production deployments.

`Details of how each prerequisite/dependency is quickly satisfied:`    
* Operating System Prerequisite: Any Linux distribution that supports Docker should work.
* Operating System Pre-configuration: This quick start includes easy paste-able commands to quickly satisfy this prerequisite.
* Kubernetes Cluster Prerequisite: is implemented using k3d (k3s in Docker)
* Default Storage Class Prerequisite: k3d ships with a local volume storage class.
* Support for automated provisioning of Kubernetes Service of type LB Prerequisite: is implemented by taking advantage of k3d's ability to easily map port 443 of the VM to port 443 of a Dockerized LB that forwards traffic to a single Istio Ingress Gateway.
Important limitations of this quick start guide's implementation of k3d to be aware of:
  * Multiple Ingress Gateways aren't supported by this implementation as they would each require their own LB, and this trick of using the host's port 443 only works for automated provisioning of a single service of type LB that leverages port 443.
  * Multiple Ingress Gateways makes a demoable/tinkerable KeyCloak and locally hosted SSO deployment much easier.
  * Multiple Ingress Gateways can be demoed on k3d if configuration tweaks are made, MetalLB is used, and you are developing using a local Linux Desktop. (network connectivity limitations of the implementation would only allow a the web browser on the k3d host server to see the webpages.)
  * If you want to easily demo and tinker with Multiple Ingress Gateways and Keycloak, then MetalLB + k3s (or another non-Dockerized Kubernetes distribution) would be a happy path to look into. (or alternatively create an issue ticket requesting prioritization of a keycloak quick start or better yet a Merge Request.)
* Access to Container Images Prerequisite is satisfied by using personal image pull credentials and internet connectivity to <registry1.dso.mil>
* Customer Controlled Private Git Repo Prerequisite isn't required due to substituting declarative git ops installation of the Big Bang Helm chart with an imperative helm cli based installation.
* Encrypting Secrets as code Prerequisite is substituted with clear text secrets on your local machine.
* Installing and Configuring Flux Prerequisite: Not using GitOps for the quick start eliminates the need to configure flux, and installation is covered within this guide.
* HTTPS Certificate and hostname configuration Prerequisites: Are satisfied by leveraging default hostname values and the demo HTTPS wildcard certificate that's uploaded to the Big Bang repo, which is valid for *.bigbang.dev,*.admin.bigbang.dev, and a few others. The demo HTTPS wildcard certificate is signed by the Lets Encrypt Free, a Certificate Authority trusted on the public internet, so demo sites like grafana.bigbang.dev will show a trusted HTTPS certificate.
* DNS Prerequisite: is substituted by making use of your workstation's Hosts file.

## Step 1: Provision a Virtual Machine

The following requirements are recommended for Demonstration Purposes:

* 1 Virtual Machine with 32GB RAM, 8-Core CPU (t3a.2xlarge for AWS users), and 100GB of disk space should be sufficient.
* Ubuntu Server 20.04 LTS (Ubuntu comes up slightly faster than CentOS, in reality any Linux distribution with Docker installed should work)
* Most Cloud Service Provider provisioned VMs default to passwordless sudo being preconfigured, but if you're doing local development or a bare metal deployment then it's recommended that you configure passwordless sudo. 
  * Steps for configuring passwordless sudo: [(source)](https://unix.stackexchange.com/questions/468416/setting-up-passwordless-sudo-on-linux-distributions)
    1. `sudo visudo`
    1. Change:
       ```text
       # Allow members of group sudo to execute any command
       %sudo   ALL=(ALL:ALL) ALL
       ```
       To:
       ```text
       # Allow members of group sudo to execute any command, no password
       %sudo   ALL=(ALL:ALL) NOPASSWD:ALL
       ```
* Network connectivity to Virtual Machine (provisioning with a public IP and a security group locked down to your IP should work. Otherwise a Bare Metal server or even a Vagrant Box Virtual Machine configured for remote ssh works fine.)

> Note: If your workstation has Docker, sufficient compute, and has ports 80, 443, and 6443 free, you can use your workstation in place of a remote virtual machine and do local development.

## Step 2: SSH to Remote VM

* ssh and passwordless sudo should be configured on the remote machine
* You can skip this step if you are doing local development.

1. Setup SSH

    ```shell
    # [admin@Unix_Laptop:~]
    mkdir -p ~/.ssh
    chmod 700 ~/.ssh
    touch ~/.ssh/config
    chmod 600 ~/.ssh/config
    temp="""##########################
    Host k3d
      Hostname x.x.x.x  #IP Address of k3d node
      IdentityFile ~/.ssh/bb-onboarding-attendees.ssh.privatekey   #ssh key authorized to access k3d node
      User ubuntu
      StrictHostKeyChecking no   #Useful for vagrant where you'd reuse IP from repeated tear downs
    #########################"""
    echo "$temp" | sudo tee -a ~/.ssh/config  #tee -a, appends to preexisting config file
    ```

1. SSH to instance

    ```shell
    # [admin@Laptop:~]
    ssh k3d

    # [ubuntu@Ubuntu_VM:~]
    ```

## Step 3: Install Prerequisite Software

Note: This guide follows the DevOps best practice of left-shifting feedback on mistakes and surfacing errors as early in the process as possible. This is done by leveraging tests and verification commands.

1. Install Git

    ```shell
    sudo apt install git -y
    ```

1. Install Docker and add $USER to Docker group.

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    sudo apt update -y && sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release -y && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg && echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null && sudo apt update -y && sudo apt install docker-ce docker-ce-cli containerd.io -y && sudo usermod --append --groups docker $USER


    # Alternative command (less safe due to curl | bash, but more generic):
    # curl -fsSL https://get.docker.com | bash && sudo usermod --append --groups docker $USER
    ```

1. Logout and login to allow the `usermod` change to take effect.

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    exit
    ```

    ```shell
    # [admin@Laptop:~]
    ssh k3d
    ```

1. Verify Docker Installation

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    docker run hello-world
    ```

    ```console
    Hello from Docker!
    ```

1. Install k3d 

    > Note: k3d v4.4.8 has integration issues with Big Bang, v4.4.7 is known to work.

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    # The following downloads the 64 bit linux version of k3d v4.4.7, checks it 
    # against a copy of the sha256 checksum, if they match k3d gets installed
    wget -q -O - https://github.com/rancher/k3d/releases/download/v4.4.7/k3d-linux-amd64 > k3d

    echo 51731ffb2938c32c86b2de817c7fbec8a8b05a55f2e4ab229ba094f5740a0f60 k3d | sha256sum -c | grep OK
    # 51731ffb2938c32c86b2de817c7fbec8a8b05a55f2e4ab229ba094f5740a0f60 came from
    # wget -q -O - https://github.com/rancher/k3d/releases/download/v4.4.7/sha256sum.txt | grep k3d-linux-amd64 | cut -d ' ' -f 1

    if [ $? == 0 ]; then chmod +x k3d && sudo mv k3d /usr/local/bin/k3d; fi


    # Alternative command (less safe due to curl | bash, but more generic):
    # wget -q -O - https://raw.githubusercontent.com/rancher/k3d/main/install.sh | TAG=v4.4.7 bash
    ```

1. Verify k3d installation

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    k3d --version
    ```

    ```console
    k3d version v4.4.7
    k3s version v1.21.2-k3s1 (default)
    ```

1. Install kubectl

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    # The following downloads the 64 bit linux version of kubectl v1.22.1, checks it 
    # against a copy of the sha256 checksum, if they match kubectl gets installed
    wget -q -O - https://dl.k8s.io/release/v1.22.1/bin/linux/amd64/kubectl > kubectl

    echo 78178a8337fc6c76780f60541fca7199f0f1a2e9c41806bded280a4a5ef665c9 kubectl | sha256sum -c | grep OK
    # 78178a8337fc6c76780f60541fca7199f0f1a2e9c41806bded280a4a5ef665c9 came from
    # wget -q -O - https://dl.k8s.io/release/v1.22.1/bin/linux/amd64/kubectl.sha256

    if [ $? == 0 ]; then chmod +x kubectl && sudo mv kubectl /usr/local/bin/kubectl; fi

    # Create a symbolic link from k to kubectl
    sudo ln -s /usr/local/bin/kubectl /usr/local/bin/k
    ```

1. Verify kubectl installation

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    kubectl version --client
    ```

    ```console
    Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.1", GitCommit:"632ed300f2c34f6d6d15ca4cef3d3c7073412212", GitTreeState:"clean", BuildDate:"2021-08-19T15:45:37Z", GoVersion:"go1.16.7", Compiler:"gc", Platform:"linux/amd64"}
    ```

1. Install Kustomize

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    # The following downloads the 64 bit linux version of kustomize v4.3.0, checks it 
    # against a copy of the sha256 checksum, if they match kustomize gets installed
    wget -q -O - https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv4.3.0/kustomize_v4.3.0_linux_amd64.tar.gz > kustomize.tar.gz

    echo d34818d2b5d52c2688bce0e10f7965aea1a362611c4f1ddafd95c4d90cb63319 kustomize.tar.gz | sha256sum -c | grep OK
    # d34818d2b5d52c2688bce0e10f7965aea1a362611c4f1ddafd95c4d90cb63319
    # came from https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv4.3.0/checksums.txt

    if [ $? == 0 ]; then tar -xvf kustomize.tar.gz && chmod +x kustomize && sudo mv kustomize /usr/local/bin/kustomize && rm kustomize.tar.gz ; fi    


    # Alternative commands (less safe due to curl | bash, but more generic):
    # curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
    # chmod +x kustomize
    # sudo mv kustomize /usr/bin/kustomize
    ```

1. Verify Kustomize installation

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    kustomize version
    ```

    ```console
    {Version:kustomize/v4.3.0 GitCommit:cd17338759ef64c14307991fd25d52259697f1fb BuildDate:2021-08-24T19:24:28Z GoOs:linux GoArch:amd64}
    ```

1. Install Helm

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    # The following downloads the 64 bit linux version of helm v3.6.3, checks it 
    # against a copy of the sha256 checksum, if they match helm gets installed
    wget -q -O - https://get.helm.sh/helm-v3.6.3-linux-amd64.tar.gz > helm.tar.gz

    echo 07c100849925623dc1913209cd1a30f0a9b80a5b4d6ff2153c609d11b043e262 helm.tar.gz | sha256sum -c | grep OK
    # 07c100849925623dc1913209cd1a30f0a9b80a5b4d6ff2153c609d11b043e262
    # came from https://github.com/helm/helm/releases/tag/v3.6.3

    if [ $? == 0 ]; then tar -xvf helm.tar.gz && chmod +x linux-amd64/helm && sudo mv linux-amd64/helm /usr/local/bin/helm && rm -rf linux-amd64 && rm helm.tar.gz ; fi    


    # Alternative command (less safe due to curl | bash, but more generic):
    # curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
    ```

1. Verify Helm installation

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    helm version
    ```

    ```console
    version.BuildInfo{Version:"v3.6.3", GitCommit:"d506314abfb5d21419df8c7e7e68012379db2354", GitTreeState:"dirty", GoVersion:"go1.16.5"}
    ```

## Step 4: Configure Host Operating System Prerequisites

* Run Operating System Pre-configuration

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    # ECK implementation of ElasticSearch needs the following or will see OOM errors
    sudo sysctl -w vm.max_map_count=524288

    # SonarQube host OS pre-requisites
    sudo sysctl -w fs.file-max=131072
    ulimit -n 131072
    ulimit -u 8192

    # Needed for ECK to run correctly without OOM errors
    echo 'vm.max_map_count=524288' > /etc/sysctl.d/vm-max_map_count.conf

    # Needed by Sonarqube
    echo 'fs.file-max=131072' > /etc/sysctl.d/fs-file-max.conf

    # Load updated configuration
    sysctl --load

    # Alternative form of above 3 commands:
    # sudo sysctl -w vm.max_map_count=524288
    # sudo sysctl -w fs.file-max=131072

    # Needed by Sonarqube
    ulimit -n 131072
    ulimit -u 8192

    # Preload kernel modules required by istio-init, required for SELinux enforcing instances using istio-init
    modprobe xt_REDIRECT
    modprobe xt_owner
    modprobe xt_statistic

    # Persist modules after reboots
    printf "xt_REDIRECT\nxt_owner\nxt_statistic\n" | sudo tee -a /etc/modules

    # Kubernetes requires swap disabled
    # Turn off all swap devices and files (won't last reboot)
    sudo swapoff -a

    # For swap to stay off, you can remove any references found via
    # cat /proc/swaps
    # cat /etc/fstab
    ```

## Step 5:  Create a k3d Cluster

After reading the notes on the purpose of k3d's command flags, you will be able to copy and paste the command to create a k3d cluster.

### Explanation of k3d Command Flags, Relevant to the Quick Start

* `SERVER_IP="10.10.16.11"` and `--k3s-server-arg "--tls-san=$SERVER_IP"`:    
  These associate an extra IP to the Kubernetes API server's generated HTTPS certificate.    

  **Explanation of the effect:**

   1. If you are running k3d from a local host or you plan to run 100% of kubectl commands while ssh'd into the k3d server, then you can omit these flags or paste unmodified incorrect values with no ill effect.

   1. If you plan to run k3d on a remote server, but run kubectl, helm, and kustomize commands from a workstation, which would be needed if you wanted to do something like kubectl port-forward then you would need to specify the remote server's public or private IP address here. After pasting the ~/.kube/config file from the k3d server to your workstation, you will need to edit the IP inside of the file from 0.0.0.0 to the value you used for SERVER_IP.

  **Tips for looking up the value to plug into SERVER_IP:**

  * Method 1: If your k3d server is a remote box, then run the following command from your workstation.    
    `cat ~/.ssh/config | grep k3d -A 6`
  * Method 2: If the remote server was provisioned with a Public IP, hen run the following command from the server hosting k3d.    
    `curl ifconfig.me --ipv4`
  * Method 3: If the server hosting k3d only has a Private IP,then run the following command from the server hosting k3d    
    `ip address`    
    (You will see more than one address, use the one in the same subnet as your workstation)    

* `--volume /etc/machine-id:/etc/machine-id`:    
This is required for fluentbit log shipper to work.

* `IMAGE_CACHE=${HOME}/.k3d-container-image-cache`, `cd ~`, `mkdir -p ${IMAGE_CACHE}`, and `--volume ${IMAGE_CACHE}:/var/lib/rancher/k3s/agent/containerd/io.containerd.content.v1.content`:    
These make it so that if you fully deploy Big Bang and then want to reset the cluster to a fresh state to retest some deployment logic. Then after running `k3d cluster delete k3s-default` and redeploying, subsequent redeployments will be faster because all container images used will have been prefetched.

* `--servers 1 --agents 3`:     
These flags are not used and shouldn't be added. This is because the image caching logic works more reliably on a one node Dockerized cluster, vs a four node Dockerized cluster. If you need to add these flags to simulate multi nodes to test pod and node affinity rules, then you should remove the image cache flags, or you may experience weird image pull errors.

* `--port 80:80@loadbalancer` and `--port 443:443@loadbalancer`:    
These map the virtual machine's port 80 and 443 to port 80 and 443 of a Dockerized LB that will point to the NodePorts of the Dockerized k3s node.

### k3d Cluster Creation Commands

```shell
# [ubuntu@Ubuntu_VM:~]
SERVER_IP="10.10.16.11" #(Change this value, if you need remote kubectl access)

# Create image cache directory
IMAGE_CACHE=${HOME}/.k3d-container-image-cache

mkdir -p ${IMAGE_CACHE}

k3d cluster create \
    --k3s-server-arg "--tls-san=$SERVER_IP" \
    --volume /etc/machine-id:/etc/machine-id \
    --volume ${IMAGE_CACHE}:/var/lib/rancher/k3s/agent/containerd/io.containerd.content.v1.content \
    --k3s-server-arg "--disable=traefik" \
    --port 80:80@loadbalancer \
    --port 443:443@loadbalancer \
    --api-port 6443
```

### k3d Cluster Verification Command

```shell
# [ubuntu@Ubuntu_VM:~]
kubectl config use-context k3d-k3s-default
kubectl get node
```

```console
Switched to context "k3d-k3s-default".
NAME                       STATUS   ROLES                  AGE   VERSION
k3d-k3s-default-server-0   Ready    control-plane,master   11m   v1.21.3+k3s1
```

## Step 6: Verify Your IronBank Image Pull Credentials

1. Here we continue to follow the DevOps best practice of enabling early left-shifted feedback whenever possible; Before adding credentials to a configuration file and not finding out there is an issue until after we see an ImagePullBackOff error during deployment, we will do a quick left-shifted verification of the credentials.

1. Look up your IronBank image pull credentials
    1. In a web browser go to [https://registry1.dso.mil](https://registry1.dso.mil)
    1. Login via OIDC provider
    1. In the top right of the page, click your name, and then User Profile
    1. Your image pull username is labeled "Username"
    1. Your image pull password is labeled "CLI secret"
      > Note: The image pull credentials are tied to the life cycle of an OIDC token which expires after ~3 days, so if 3 days have passed since your last login to IronBank, the credentials will stop working until you re-login to the [https://registry1.dso.mil](https://registry1.dso.mil) GUI

1. Verify your credentials work

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    # Turn off bash history
    set +o history

    export REGISTRY1_USERNAME=<REPLACE_ME>
    export REGISTRY1_PASSWORD=<REPLACE_ME>
    echo $REGISTRY1_PASSWORD | docker login registry1.dso.mil --username $REGISTRY1_USERNAME --password-stdin
    
    # Turn on bash history
    set -o history
    ```

## Step 7: Clone your desired version of the Big Bang Umbrella Helm Chart

```shell
# [ubuntu@Ubuntu_VM:~]
cd ~
git clone https://repo1.dso.mil/platform-one/big-bang/bigbang.git
cd ~/bigbang

# Checkout version 1.15.0 of Big Bang
# (Pinning to specific versions is a DevOps best practice)
git checkout tags/1.15.0
git status
```

```console
HEAD detached at 1.15.0
```

> HEAD is git speak for current context within a tree of commits

## Step 8: Install Flux

* The `echo $REGISTRY1_USERNAME` is there to verify the value of your environmental variable is still populated. If you switch terminals or re-login, you may need to reestablish these variables.

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    echo $REGISTRY1_USERNAME
    cd ~/bigbang
    $HOME/bigbang/scripts/install_flux.sh -u $REGISTRY1_USERNAME -p $REGISTRY1_PASSWORD
    # NOTE: After running this command the terminal may appear to be stuck on
    # "networkpolicy.networking.k8s.io/allow-webhooks created"
    # It's not stuck, the end of the .sh script has a kubectl wait command, give it 5 min
    # Also if you have slow internet/hardware you might see a false error message
    # error: timed out waiting for the condition on deployments/helm-controller
    
    # As long as the following command shows STATUS Running you're good to move on
    kubectl get pods --namespace=flux-system
    ```
    
    ```console
    NAME                                     READY   STATUS    RESTARTS   AGE
    kustomize-controller-d689c6688-bnr96     1/1     Running   0          3m8s
    notification-controller-65dffcb7-zk796   1/1     Running   0          3m8s
    source-controller-5fdb69cc66-g5dlh       1/1     Running   0          3m8s
    helm-controller-6c67b58f78-cvxmv         1/1     Running   0          3m8s
    ```

## Step 9: Create helm values .yaml files to act as input variables for the Big Bang Helm Chart

> Note for those new to linux: The following are multi line copy pasteable commands to quickly generate config files from the CLI, make sure you copy from cat to EOF, if you get stuck in the terminal use ctrl + c

```shell
# [ubuntu@Ubuntu_VM:~]
cat << EOF > ~/ib_creds.yaml
registryCredentials:
  registry: registry1.dso.mil
  username: "$REGISTRY1_USERNAME"
  password: "$REGISTRY1_PASSWORD"
EOF


cat << EOF > ~/demo_values.yaml
logging:
  values:
    kibana:
      count: 1
      resources:
        requests:
          cpu: 400m
          memory: 1Gi
        limits:
          cpu: null  # nonexistent cpu limit results in faster spin up
          memory: null
    elasticsearch:
      master:
        count: 1
        resources:
          requests:
            cpu: 400m
            memory: 2Gi
          limits:
            cpu: null
            memory: null
      data:
        count: 1
        resources:
          requests:
            cpu: 400m
            memory: 2Gi
          limits: 
            cpu: null
            memory: null

clusterAuditor:
  values:
    resources:
      requests:
        cpu: 400m
        memory: 2Gi
      limits:
        cpu: null
        memory: null

gatekeeper:
  enabled: false
  values:
    replicas: 1
    controllerManager:
      resources:
        requests:
          cpu: 100m
          memory: 512Mi
        limits:
          cpu: null
          memory: null
    audit:
      resources:
        requests:
          cpu: 400m
          memory: 768Mi
        limits:
          cpu: null
          memory: null
    violations:
      allowedDockerRegistries:
        enforcementAction: dryrun

istio:
  values:
    values: # possible values found here https://istio.io/v1.5/docs/reference/config/installation-options (ignore 1.5, latest docs point here)
      global: # global istio operator values
        proxy: # mutating webhook injected istio sidecar proxy's values
          resources:
            requests:
              cpu: 0m # null get ignored if used here
              memory: 0Mi
            limits:
              cpu: 0m
              memory: 0Mi

twistlock:
  enabled: false # twistlock requires a license to work, so we're disabling it
EOF
```

## Step 10: Install Big Bang using the local development workflow

```shell
# [ubuntu@Ubuntu_VM:~]
helm upgrade --install bigbang $HOME/bigbang/chart \
  --values $HOME/bigbang/chart/ingress-certs.yaml \
  --values $HOME/ib_creds.yaml \
  --values $HOME/demo_values.yaml \
  --namespace=bigbang --create-namespace
```

Explanation of flags used in the imperative helm install command:

`upgrade --install`
: This makes the command more idempotent by allowing the exact same command to work for both the initial installation and upgrade use cases.

`bigbang $HOME/bigbang/chart`
: bigbang is the name of the helm release that you'd see if you run `helm list -n=bigbang`. `$HOME/bigbang/chart` is a reference to the helm chart being installed.

`--values $HOME/bigbang/chart/ingress-certs.yaml`
: References demonstration HTTPS certificates embedded in the public repository. The *.bigbang.dev wildcard certificate is signed by Let's Encrypt, a free public internet Certificate Authority.

`--namespace=bigbang --create-namespace`
: Means it will install the bigbang helm chart in the bigbang namespace and create the namespace if it doesn't exist.


## Step 11: Verify Big Bang has had enough time to finish installing

* If you try to run the command in Step 12 too soon, you'll see an ignorable temporary error message

    ```shell
    # [ubuntu@Ubuntu_VM:~]
    kubectl get virtualservices --all-namespaces
    
    # Note after running the above command, you may see an ignorable temporary error message
    # The error message may be different based on your timing, but could look like this:
    #     error: the server doesn't have a resource type "virtualservices"
    #     or
    #     No resources found
    
    # The above errors could be seen if you run the command too early 
    # Give Big Bang some time to finish installing, then run the following command to check it's status
    
    k get po -A
    ```

* If after running `k get po -A` (which is the shorthand of `kubectl get pods --all-namespaces`) you see something like the following, then you need to wait longer

    ```console
    NAMESPACE           NAME                                                READY   STATUS              RESTARTS   AGE
    kube-system         metrics-server-86cbb8457f-dqsl5                     1/1     Running             0          39m
    kube-system         coredns-7448499f4d-ct895                            1/1     Running             0          39m
    flux-system         notification-controller-65dffcb7-qpgj5              1/1     Running             0          32m
    flux-system         kustomize-controller-d689c6688-6dd5n                1/1     Running             0          32m
    flux-system         source-controller-5fdb69cc66-s9pvw                  1/1     Running             0          32m
    kube-system         local-path-provisioner-5ff76fc89d-gnvp4             1/1     Running             1          39m
    flux-system         helm-controller-6c67b58f78-6dzqw                    1/1     Running             0          32m
    gatekeeper-system   gatekeeper-controller-manager-5cf7696bcf-xclc4      0/1     Running             0          4m6s
    gatekeeper-system   gatekeeper-audit-79695c56b8-qgfbl                   0/1     Running             0          4m6s
    istio-operator      istio-operator-5f6cfb6d5b-hx7bs                     1/1     Running             0          4m8s
    eck-operator        elastic-operator-0                                  1/1     Running             1          4m10s
    istio-system        istiod-65798dff85-9rx4z                             1/1     Running             0          87s
    istio-system        public-ingressgateway-6cc4dbcd65-fp9hv              0/1     ContainerCreating   0          46s
    logging             logging-fluent-bit-dbkxx                            0/2     Init:0/1            0          44s
    monitoring          monitoring-monitoring-kube-admission-create-q5j2x   0/1     ContainerCreating   0          42s
    logging             logging-ek-kb-564d7779d5-qjdxp                      0/2     Init:0/2            0          41s
    logging             logging-ek-es-data-0                                0/2     Init:0/2            0          44s
    istio-system        svclb-public-ingressgateway-ggkvx                   5/5     Running             0          39s
    logging             logging-ek-es-master-0                              0/2     Init:0/2            0          37s
    ```

* Wait up to 10 minutes then re-run `k get po -A`, until all pods show STATUS Running

* `helm list -n=bigbang` should also show STATUS deployed

    ```console
    NAME                         	NAMESPACE        	REVISION	UPDATED                                	STATUS  	CHART                            	APP VERSION
    bigbang                      	bigbang          	1       	2021-08-31 16:50:39.336392871 +0000 UTC	deployed	bigbang-1.15.0
    eck-operator-eck-operator    	eck-operator     	1       	2021-08-31 16:21:12.546012077 +0000 UTC	deployed	eck-operator-1.6.0-bb.2          	1.6.0
    gatekeeper-system-gatekeeper 	gatekeeper-system	1       	2021-08-31 16:21:13.146595333 +0000 UTC	deployed	gatekeeper-3.5.1-bb.16           	v3.5.1
    istio-operator-istio-operator	istio-operator   	1       	2021-08-31 16:21:12.726676226 +0000 UTC	deployed	istio-operator-1.9.7-bb.1
    istio-system-istio           	istio-system     	1       	2021-08-31 16:44:07.776386128 +0000 UTC	deployed	istio-1.9.7-bb.0
    jaeger-jaeger                	jaeger           	1       	2021-08-31 16:25:17.733322853 +0000 UTC	deployed	jaeger-operator-2.23.0-bb.1      	1.24.0
    kiali-kiali                  	kiali            	1       	2021-08-31 16:25:14.314905637 +0000 UTC	deployed	kiali-operator-1.37.0-bb.3       	1.37.0
    logging-cluster-auditor      	logging          	1       	2021-08-31 16:25:33.628134776 +0000 UTC	deployed	cluster-auditor-0.3.0-bb.6       	1.16.0
    logging-ek                   	logging          	1       	2021-08-31 16:22:12.609559643 +0000 UTC	deployed	logging-0.1.20-bb.0              	7.13.4
    logging-fluent-bit           	logging          	1       	2021-08-31 16:22:41.467862784 +0000 UTC	deployed	fluent-bit-0.16.1-bb.0           	1.8.1
    monitoring-monitoring        	monitoring       	1       	2021-08-31 16:22:26.03075708 +0000 UTC 	deployed	kube-prometheus-stack-14.0.0-bb.8	0.46.0
    ```

## Step 12: Edit your workstation's Hosts file to access the web pages hosted on the Big Bang Cluster

Run the following command, which is the short hand equivalent of `kubectl get virtualservices --all-namespaces` to see a list of websites you'll need to add to your hosts file

```shell
k get vs -A
```

```console
NAMESPACE    NAME                                      GATEWAYS                  HOSTS                          AGE
logging      kibana                                    ["istio-system/public"]   ["kibana.bigbang.dev"]         38m
monitoring   monitoring-monitoring-kube-grafana        ["istio-system/public"]   ["grafana.bigbang.dev"]        36m
monitoring   monitoring-monitoring-kube-alertmanager   ["istio-system/public"]   ["alertmanager.bigbang.dev"]   36m
monitoring   monitoring-monitoring-kube-prometheus     ["istio-system/public"]   ["prometheus.bigbang.dev"]     36m
kiali        kiali                                     ["istio-system/public"]   ["kiali.bigbang.dev"]          35m
jaeger       jaeger                                    ["istio-system/public"]   ["tracing.bigbang.dev"]        35m
```

### Linux/Mac Users

```shell
# [admin@Laptop:~]
sudo vi /etc/hosts
```

### Windows Users

1. Right click Notepad -> Run as Administrator
1. Open C:\Windows\System32\drivers\etc\hosts

### Linux/Mac/Windows Users

Add the following entries to the Hosts file, where x.x.x.x = k3d virtual machine's IP.

> Hint: find and replace is your friend

```plaintext
x.x.x.x  kibana.bigbang.dev
x.x.x.x  grafana.bigbang.dev
x.x.x.x  alertmanager.bigbang.dev
x.x.x.x  prometheus.bigbang.dev
x.x.x.x  kiali.bigbang.dev
x.x.x.x  tracing.bigbang.dev
x.x.x.x  argocd.bigbang.dev
```

## Step 13: Visit a webpage

In a browser, visit one of the sites listed using the `k get vs -A` command

## Step 14: Play

Here's an example of post deployment customization of Big Bang.
After looking at <https://repo1.dso.mil/platform-one/big-bang/bigbang/-/blob/master/chart/values.yaml>
It should make sense that the following is a valid edit

```shell
# [ubuntu@Ubuntu_VM:~]

cat << EOF > ~/tinkering.yaml
addons:
  argocd:
    enabled: true
EOF

helm upgrade --install bigbang $HOME/bigbang/chart \
--values $HOME/bigbang/chart/ingress-certs.yaml \
--values $HOME/ib_creds.yaml \
--values $HOME/demo_values.yaml \
--values $HOME/tinkering.yaml \
--namespace=bigbang --create-namespace

# NOTE: There may be a ~1 minute delay for the change to apply

k get vs -A
# Now ArgoCD should show up, if it doesn't wait a minute and rerun the command

k get po -n=argocd
# Once these are all Running you can visit argocd's webpage
```

> Remember to un-edit your Hosts file when you are finished tinkering.
