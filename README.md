# gitops-repo-wedevs
# k3s + FluxCD + SOPS(age) + MetalLB + ingress-nginx + WordPress


Goal (Task 1):

Create a local k3s cluster.

Bootstrap FluxCD and configure GitOps to sync from a GitHub repo.

Use SOPS with age to encrypt secrets, and allow Flux to decrypt them in-cluster.

Deploy a bare-metal load balancer (MetalLB) and configure an AddressPool.

Deploy ingress-nginx via Helm (chart 4.11.7) using Flux and expose it as a LoadBalancer.

Deploy MySQL and WordPress via Flux, with MySQL credentials encrypted using SOPS+age in the Git repository.

After following the steps below, WordPress should be reachable at http://app.local (you will add an /etc/hosts entry pointing to the MetalLB service IP).



## Assumptions & choices made

This guide favors k3d (k3s-in-Docker) for local convenience. If you prefer installing k3s directly on the host, skip the k3d commands and install k3s with the upstream installer.

IP range for MetalLB: 10.0.0.240-10.0.0.250. If this conflicts on your machine, adjust it.

GitHub repo owner/repo/branch/path are placeholders — replace with your values.

Flux v2 (GitOps Toolkit) is used.



## Prerequisites

Docker installed and running

k3d installed

kubectl installed

flux CLI installed

sops and age (age-keygen) installed

A GitHub repository where Flux will read manifests (or create one)


Solution:

1. Install k3d
K3d is a lightweight wrapper to run k3s, a minimal Kubernetes distribution, in Docker. It's great for local development.

First, make sure you have Docker installed and running on your system. If you don't, you can install it using:

Bash

sudo apt update
sudo apt install docker.io -y
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
After running the usermod command, you'll need to log out and log back in for the changes to take effect.

Once Docker is ready, you can install k3d with their official install script:

Bash

wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
2. Install kubectl
kubectl is the command-line tool for interacting with a Kubernetes cluster. The recommended way to install it is using apt.

First, update the apt package index and install the necessary dependencies:

Bash

sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl
Next, add the official Kubernetes repository GPG key:

Bash

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
Then, add the Kubernetes apt repository:

Bash

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
Finally, update the apt package index with the new repository and install kubectl:

Bash

sudo apt update
sudo apt install -y kubectl


3. Install Flux CLI

FluxCD provides a reliable install script that handles versioning, architecture, and placement for you.

Run this:

bash
curl -s https://fluxcd.io/install.sh | sudo bash
This script:

Detects your OS and architecture

Downloads the correct binary

Installs it to /usr/local/bin/flux

Then verify:

bash
flux --version


4. Install sops and age (age-keygen)
sops (Secrets Operations) is a tool for managing encrypted secrets. age is a simple, modern and secure file encryption tool. age-keygen is a utility that comes with age for generating encryption keys.

First, install age. The easiest way is to download the precompiled binary and place it in your PATH.


# Set the version you want
version="v1.1.1"

# Download the correct tarball
curl -sL -o age.tar.gz "https://github.com/FiloSottile/age/releases/download/${version}/age-${version}-linux-amd64.tar.gz"

# Verify it's a gzip archive
if file age.tar.gz | grep -q 'gzip compressed'; then
    tar -zxvf age.tar.gz
    sudo mv age/age age/age-keygen /usr/local/bin/
    echo "✅ age CLI installed successfully"
else
    echo "❌ Download failed or redirected. Not a gzip file."
    exit 1
fi

The above commands download the age release, extract the binaries, and move both age and age-keygen to /usr/local/bin.


Next, install sops. There isn't a direct apt package for sops, so you'll also download its binary.


Bash

curl -sL https://github.com/mozilla/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64 | sudo tee /usr/local/bin/sops > /dev/null
sudo chmod +x /usr/local/bin/sops
This command downloads the sops binary, makes it executable, and places it in /usr/local/bin.







1) Create a local k3s cluster using k3d

# create a k3d cluster with 1 server and 1 agent and a metallb-friendly network
k3d cluster create mycluster --servers 1 --agents 1 \
--api-port 6550 -p "8080:80@loadbalancer" -p "8443:443@loadbalancer"

# set kubeconfig (k3d does this for you but ensure kubectl context)
kubectl cluster-info --context k3d-mycluster


2) Generate an age keypair for SOPS

# generate age keypair
age-keygen -o ./age-key.txt
# public key is in the file; example public key line begins with 'age1...'
cat ./age-key.txt

Keep age-key.txt safe. You'll use the public key to encrypt files for the repo and the private-age key will be stored as a Kubernetes Secret in the flux-system namespace so that Flux controllers can decrypt.


age-keygen -o ./age-key.txt
Public key: age1huwd75dmxqgkxvke468f6zs64s33npfezr9j78w68e6w8d5s09jqxtcwse

cat age-key.txt
age-keygen: warning: writing secret key to a world-readable file
Public key: age1753mzkjugyltz3vd5grwrvd4kzskw9zfkzwq8zrxr6a94upnq3sqcdf2c7



3) Prepare your Git repository layout (suggested)




4) Create and encrypt the MySQL secret using SOPS + age

Create a k8s Secret manifest (plaintext) secrets/mysql-credentials.yaml:


Encrypt it with sops using the public age key (replace <AGE_PUBLIC_KEY> with the actual public key or the file containing it):

# If you have a file with the age public key in $AGE_PUB
sops --encrypt --age "age1753mzkjugyltz3vd5grwrvd4kzskw9zfkzwq8zrxr6a94upnq3sqcdf2c7" --in-place secrets/mysql-credentials.yaml

The encrypted file will be a YAML containing sops: metadata and encrypted values. Commit this encrypted file to the secrets/ folder in your Git repo.


5) Put the age private key into the cluster for Flux (so flux controllers can decrypt)

Flux's controllers (kustomize-controller or sops-controller) expect an in-cluster secret with the age private key. Create a secret in flux-system named sops-age with the key file stored under age.agekey (this exact key name is recognized by the integration):
Keep the age-key.txt file to project directory and run below command:

kubectl create ns flux-system || true
kubectl -n flux-system create secret generic sops-age --from-file=age.agekey=./secrets/age-key.txt

kubectl -n flux-system create secret generic sops-age-key --from-file=age.agekey=./clusters/mycluster/secrets/age-key.txt


References: Flux SOPS guide recommends creating sops-age secret with the age key so the Kustomize/Flux controllers can decrypt. 
(Flux docs & community examples follow this pattern.)



6) Bootstrap Flux into the cluster and point it at your GitHub repo

Use the Flux CLI bootstrap github command — replace placeholders with your GitHub --owner, --repository, and --path values. Example:

flux bootstrap github \
--owner=helalcse2k \
--repository=gitops-repo-wedevs \
--branch=main \
--path=clusters/mycluster \
--personal


7) Deploy MetalLB (HelmRelease)

Create apps/metallb/helmrepository.yaml and apps/metallb/helmrelease.yaml under your repo. 
Example HelmRepository for the official MetalLB chart:

kubectl create ns metallb-system

# apps/metallb/helmrepository.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
    name: metallb
    namespace: flux-system
spec:
    interval: 10m
    url: https://metallb.github.io/metallb


# apps/metallb/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
    name: metallb
    namespace: metallb-system
spec:
    chart:
        spec:
        chart: metallb
        version: "0.13.10" # pick a stable chart version or omit
        sourceRef:
            kind: HelmRepository
            name: metallb
            namespace: flux-system
    interval: 5m
    install:
    createNamespace: true


After MetalLB is installed, add the address pool CRs into your repo (e.g., apps/metallb/address-pool.yaml):

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
    name: default-addresspool
    namespace: metallb-system
spec:
    addresses:
    - 10.0.0.240-10.0.0.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
    name: l2
    namespace: metallb-system
spec: {}

When MetalLB is up the LoadBalancer services can get an IP from that pool.


8) Install ingress-nginx via Flux as a HelmRelease (chart version 4.11.7)

Create a HelmRepository for the ingress-nginx chart:

# apps/ingress-nginx/helmrepo.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
    name: ingress-nginx
    namespace: flux-system
spec:
    url: https://kubernetes.github.io/ingress-nginx
    interval: 1m

Create a HelmRelease to install the chart as a LoadBalancer service:

kubectl create ns ingress-nginx

# apps/ingress-nginx/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
    name: ingress-nginx
    namespace: ingress-nginx
spec:
    chart:
        spec:
            chart: ingress-nginx
            version: "4.11.7"
            sourceRef:
                kind: HelmRepository
                name: ingress-nginx
                namespace: flux-system
    interval: 5m
    install:
        createNamespace: true
    values:
        controller:
            service:
                type: LoadBalancer
                annotations: {}
                # If you want to request a specific IP from MetalLB you can set loadBalancerIP:
                # loadBalancerIP: 10.0.0.241


This will create a LoadBalancer service for the ingress controller which MetalLB will assign an IP from the pool above. Note: If you want a specific IP, set controller.service.loadBalancerIP.


9) Deploy MySQL (HelmRelease) and point WordPress at it

We will use Bitnami's MySQL chart but we will not put plaintext credentials in the chart values — instead we will create the mysql-credentials Secret (encrypted with SOPS+age) in the wordpress namespace.

HelmRepository for Bitnami charts:

# apps/charts/bitnami-helmrepo.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
    name: bitnami
    namespace: flux-system
spec:
    url: https://charts.bitnami.com/bitnami
    interval: 1m


HelmRelease for MySQL (values configured to read from existing secret):

kubectl create ns wordpress


# apps/mysql/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
    name: mysql
    namespace: wordpress
spec:
    chart:
        spec:
            chart: mysql
            version: "14.0.0" # (example) choose compatible chart version
            sourceRef:
                kind: HelmRepository
                name: bitnami
                namespace: flux-system
    interval: 5m
    install:
        createNamespace: true
    values:
        auth:
            existingSecret: mysql-credentials
        # username/password are provided by the secret 'mysql-credentials' in the wordpress namespace
        primary:
            service:
                type: ClusterIP

Important: Commit secrets/mysql-cred.yaml (SOPS-encrypted) to secrets/ in the repo so Flux can sync and decrypt it. Ensure the secret resource metadata.namespace: wordpress matches.


10) Deploy WordPress (HelmRelease) and use external DB connection

Create a HelmRelease for WordPress using the Bitnami chart and point DB settings to the MySQL service and use the secret:

# apps/wordpress/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
    name: wordpress
    namespace: wordpress
spec:
    chart:
        spec:
            chart: wordpress
            version: "25.0.0" # example, pick a stable matching chart
            sourceRef:
                kind: HelmRepository
                name: bitnami
                namespace: flux-system
    interval: 5m
    install:
        createNamespace: true
    values:
        mariadb:
            enabled: false
        externalDatabase:
            host: mysql-primary.wordpress.svc.cluster.local
            port: 3306
            user: "${MYSQL_USERNAME}" # will be read from secret via chart, see below
            password: "${MYSQL_PASSWORD}"
            database: bitnami_wordpress
        ingress:
            enabled: true
            hostname: app.local
            pathType: ImplementationSpecific
            annotations: {}


Because many charts expect raw values, prefer using the existingSecret mechanism if the chart supports it. For the bitnami wordpress chart the external DB username/password are provided by values — you can template them into the HelmRelease (not recommended to put plaintext). A better approach is to create the Kubernetes Secret mysql-credentials and patch WordPress Deployment to read creds from that Secret (or use chart's existingSecret/externalDatabase fields if available). Check the chart docs for the exact value keys.



11) Make app.local resolve to the ingress LoadBalancer IP

Get the ingress controller LoadBalancer IP after the HelmRelease is reconciled (MetalLB will assign one):

kubectl -n ingress-nginx get svc
# note EXTERNAL-IP for the ingress-nginx controller service


1. Create the Docker Hub Credentials Secret 🔑
First, create a secret in the flux-system namespace containing your Docker Hub credentials. You can do this with the kubectl create secret command. Replace YOUR_DOCKER_HUB_USERNAME and YOUR_DOCKER_HUB_PASSWORD with your actual credentials.

Bash

kubectl create secret docker-registry flux-docker-reg \
  --namespace=flux-system \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<Username> \
  --docker-password=<PAT>

 



flux reconcile source git flux-system -n flux-system

flux reconcile source helm bitnami -n flux-system

flux reconcile kustomization flux-system -n flux-system




flux reconcile helmrelease wordpress -n wordpress --with-source

flux reconcile helmrelease mysql -n wordpress --with-source


flux get helmreleases -n wordpress
kubectl -n wordpress get pods -w





kubectl create secret generic mysql-credentials \
  --namespace=wordpress \
  --from-literal=mariadb-password='your-db-password' \
  --from-literal=mariadb-user='your-db-user' \
  --from-literal=mariadb-database='bitnami_wordpress'


