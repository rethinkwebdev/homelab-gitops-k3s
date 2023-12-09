<!-- markdownlint-disable MD013 -->
# Homelab cluster with k3s and Flux

This repo configures a single Kubernetes ([k3s](https://k3s.io)) cluster with [Ansible](https://www.ansible.com) and uses the GitOps tool [Flux](https://toolkit.fluxcd.io) to manage its state.

## ✨ Features

- Automated, reproducible, customizable setup through Ansible templates and playbooks
- Opinionated implementation of Flux with [strong community support](https://github.com/onedr0p/flux-cluster-template/tree/main#-help)
- Encrypted secrets thanks to [SOPS](https://github.com/getsops/sops) and [Age](https://github.com/FiloSottile/age)
- Web application firewall thanks to [Cloudflare Tunnels](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- SSL certificates thanks to [Cloudflare](https://cloudflare.com) and [cert-manager](https://cert-manager.io)
- HA control plane capability thanks to [kube-vip](https://kube-vip.io)
- Next-gen networking thanks to [Cilium](https://cilium.io/)
- A [Renovate](https://www.mend.io/renovate)-ready repository with pull request diffs provided by [flux-local](https://github.com/allenporter/flux-local)
- Integrated [GitHub Actions](https://github.com/features/actions)

... and more!

## 📝 Prerequisites

- [ ] a domain managed on Cloudflare.
- [ ] a DNS server that supports split DNS (e.g. Pi-Hole) deployed somewhere outside your cluster **ON** your home network.

## 🚀 Installation

### 📍 Set up your local environment

1. Install the most recent version of [task](https://taskfile.dev/)

    📍 _See the task [installation docs](https://taskfile.dev/installation/) for other platforms_

    ```sh
    # Brew
    brew install go-task
    ```

2. Install the most recent version of [direnv](https://direnv.net/)

    📍 _See the direnv [installation docs](https://direnv.net/docs/installation.html) for other platforms_

    📍 _After installing `direnv` be sure to [hook it into your shell](https://direnv.net/docs/hook.html) and after that is done run `direnv allow` while in your repos directory._

    ```sh
    # Brew
    brew install direnv
    ```

3. Install the most recent version of [pipx](https://pipx.pypa.io/stable/)

    📍 _See the pipx [installation docs](https://pipx.pypa.io/stable/#install-pipx) for other platforms_

    ```sh
    # Brew
    brew install pipx
    pipx ensurepath
    pipx completions
    ```

4. Setup a install Ansible with pipx by running the following task command.

    📍 _This commands requires Python 3.10+ to be installed_

    ```sh
    # Platform agnostic
    task bootstrap:deps
    ```

5. Install the required tools: [age](https://github.com/FiloSottile/age), [flux](https://toolkit.fluxcd.io/), [cloudflared](https://github.com/cloudflare/cloudflared), [kubectl](https://kubernetes.io/docs/tasks/tools/), [sops](https://github.com/getsops/sops)

   ```sh
   # Brew
   task brew:deps
   ```

### 🔧 Do bootstrap configuration

📍 _Both `bootstrap/vars/config.yaml` and `bootstrap/vars/addons.yaml` files contain necessary information that is needed by bootstrap process._

1. Generate the `bootstrap/vars/config.yaml` and `bootstrap/vars/addons.yaml` configuration files.

    ```sh
    task bootstrap:init
    ```

2. Setup Age private / public key

    📍 _Using [SOPS](https://github.com/getsops/sops) with [Age](https://github.com/FiloSottile/age) allows us to encrypt secrets and use them in Ansible and Flux._

    2a. Create a Age private / public key (this file is gitignored)

      ```sh
      age-keygen -o age.key
      ```

    2b. Fill out the appropriate vars in `bootstrap/vars/config.yaml`

3. Create Cloudflare API Token

    📍 _To use `cert-manager` with the Cloudflare DNS challenge you will need to create a API Token._

   3a. Head over to Cloudflare and create a API Token by going [here](https://dash.cloudflare.com/profile/api-tokens).

   3b. Under the `API Tokens` section click the blue `Create Token` button.

   3c. Click the blue `Use template` button for the `Edit zone DNS` template.

   3d. Name your token something like `home-kubernetes`

   3e. Under `Permissions`, click `+ Add More` and add each permission below:

    ```text
    Zone - DNS - Edit
    Account - Cloudflare Tunnel - Read
    ```

   3f. Limit the permissions to a specific account and zone resources.

   3g. Fill out the appropriate vars in `bootstrap/vars/config.yaml`

4. Create Cloudflare Tunnel

    📍 _To expose services to the internet you will need to create a [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)._

    4a. Authenticate cloudflared to your domain

      ```sh
      cloudflared tunnel login
      ```

    4b. Create the tunnel

      ```sh
      cloudflared tunnel create k8s
      ```

    4c. In the `~/.cloudflared` directory there will be a json file with details you need. Ignore the `cert.pem` file.

    4d. Fill out the appropriate vars in `bootstrap/vars/config.yaml`

5. Complete filling out the rest of the `bootstrap/vars/config.yaml` configuration file.

    5a. Ensure `bootstrap_acme_production_enabled` is set to `false`.

    5b. [Optional] Update `bootstrap/vars/addons.yaml` and enable applications you would like included.

6. Once done run the following command which will verify and generate all the files needed to continue.

   ```sh
   task bootstrap:render
   ```

   📍 _The configure task will create a `./ansible` directory and the following directories under `./kubernetes`._

   ```sh
   📁 kubernetes      # Kubernetes cluster defined as code
   ├─📁 bootstrap     # Flux installation (not tracked by Flux)
   ├─📁 flux          # Main Flux configuration of repository
   └─📁 apps          # Apps deployed into the cluster grouped by namespace
   ```

### ⚡ Prepare your nodes for k3s

📍 _Here we will be running an Ansible playbook to prepare your nodes for running a Kubernetes cluster._

1. Ensure you are able to SSH into your nodes from your workstation using a private SSH key **without a passphrase** (for example using a SSH agent). This lets Ansible interact with your nodes.

2. Verify Ansible can view your config

    ```sh
    task ansible:list
    ```

3. Verify Ansible can ping your nodes

    ```sh
    task ansible:ping
    ```

4. Run the Ansible prepare playbook (nodes will reboot when done)

    ```sh
    task ansible:prepare
    ```

### 🛰️ Use Ansible to install k3s

📍 _Here we will be running a Ansible Playbook to install [k3s](https://k3s.io/) with [this](https://galaxy.ansible.com/xanmanning/k3s) Ansible galaxy role. If you run into problems, you can run `task ansible:nuke` to destroy the k3s cluster and start over from this point._

1. Verify Ansible can view your config

    ```sh
    task ansible:list
    ```

2. Verify Ansible can ping your nodes

    ```sh
    task ansible:ping
    ```

3. Install k3s with Ansible

    ```sh
    task ansible:install
    ```

4. Verify the nodes are online

    📍 _If this command **fails** you likely haven't configured `direnv` as mentioned previously in the guide._

    ```sh
    kubectl get nodes -o wide
    # NAME           STATUS   ROLES                       AGE     VERSION
    # k8s-0          Ready    control-plane,etcd,master   1h      v1.27.3+k3s1
    # k8s-1          Ready    worker                      1h      v1.27.3+k3s1
    ```

5. The `kubeconfig` for interacting with your cluster should have been created in the root of your repository.

### 🔹 Install Flux in your cluster

📍 _Here we will be installing [flux](https://fluxcd.io/flux/) after some quick bootstrap steps._

1. Verify Flux can be installed

    ```sh
    flux check --pre
    # ► checking prerequisites
    # ✔ kubectl 1.27.3 >=1.18.0-0
    # ✔ Kubernetes 1.27.3+k3s1 >=1.16.0-0
    # ✔ prerequisites checks passed
    ```

2. Push you changes to git

   📍 **Verify** all the `*.sops.yaml` and `*.sops.yaml` files under the `./ansible`, and `./kubernetes` directories are **encrypted** with SOPS

    ```sh
    git add -A
    git commit -m "Initial commit :rocket:"
    git push
    ```

3. Install Flux and sync the cluster to the Git repository

    ```sh
    task cluster:install
    # namespace/flux-system configured
    # customresourcedefinition.apiextensions.k8s.io/alerts.notification.toolkit.fluxcd.io created
    # ...
    ```

4. Verify Flux components are running in the cluster

    ```sh
    kubectl -n flux-system get pods -o wide
    # NAME                                       READY   STATUS    RESTARTS   AGE
    # helm-controller-5bbd94c75-89sb4            1/1     Running   0          1h
    # kustomize-controller-7b67b6b77d-nqc67      1/1     Running   0          1h
    # notification-controller-7c46575844-k4bvr   1/1     Running   0          1h
    # source-controller-7d6875bcb4-zqw9f         1/1     Running   0          1h
    ```

### ☑️ Verification Steps

1. Output all the common resources in your cluster.

    📍 _Feel free to use the provided [cluster tasks](.taskfiles/ClusterTasks.yaml) for validation of cluster resources or continue to get familiar with the `kubectl` and `flux` CLI tools._

    ```sh
    task k8s:resources
    ```

2. ⚠️ It might take `cert-manager` awhile to generate certificates, this is normal so be patient.

## 📣 Post installation

### 🌐 Public DNS

The `external-dns` application created in the `networking` namespace will handle creating public DNS records. By default, `echo-server` and the `flux-webhook` are the only subdomains reachable from the public internet. In order to make additional applications public you must set set the correct ingress class name and ingress annotations like in the HelmRelease for `echo-server`.

### 🏠 Home DNS

`k8s_gateway` will provide DNS resolution to external Kubernetes resources (i.e. points of entry to the cluster) from any device that uses your home DNS server. For this to work, your home DNS server must be configured to forward DNS queries for `${bootstrap_cloudflare_domain}` to `${bootstrap_k8s_gateway_addr}` instead of the upstream DNS server(s) it normally uses. This is a form of **split DNS** (aka split-horizon DNS / conditional forwarding).

📍 _Below is how to configure a Pi-hole for split DNS. Other platforms should be similar._

1. Apply this file on the server

   ```sh
   # /etc/dnsmasq.d/99-k8s-gateway-forward.conf
   server=/${bootstrap_cloudflare_domain}/${bootstrap_k8s_gateway_addr}
   ```

2. Restart dnsmasq on the server.

3. Query an internal-only subdomain from your workstation (any `internal` class ingresses): `dig @${home-dns-server-ip} hubble.${bootstrap_cloudflare_domain}`. It should resolve to `${bootstrap_internal_ingress_addr}`.

If you're having trouble with DNS be sure to check out these two GitHub discussions: [Internal DNS](https://github.com/onedr0p/flux-cluster-template/discussions/719) and [Pod DNS resolution broken](https://github.com/onedr0p/flux-cluster-template/discussions/635).

... Nothing working? That is expected, this is DNS after all!

#### 📜 Certificates

By default this template will deploy a wildcard certificate using the Let's Encrypt **staging environment**, which prevents you from getting rate-limited by the Let's Encrypt production servers if your cluster doesn't deploy properly (for example due to a misconfiguration). Once you are sure you will keep the cluster up for more than a few hours be sure to switch to the production servers as outlined in `config.yaml`.

📍 _You will need a production certificate to reach internet-exposed applications through `cloudflared`._

#### 🪝 Github Webhook

By default Flux will periodically check your git repository for changes. In order to have Flux reconcile on `git push` you must configure Github to send `push` events.

1. Obtain the webhook path

    📍 _Hook id and path should look like `/hook/12ebd1e363c641dc3c2e430ecf3cee2b3c7a5ac9e1234506f6f5f3ce1230e123`_

    ```sh
    kubectl -n flux-system get receiver github-receiver -o jsonpath='{.status.webhookPath}'
    ```

2. Piece together the full URL with the webhook path appended

    ```text
    https://flux-webhook.${bootstrap_cloudflare_domain}/hook/12ebd1e363c641dc3c2e430ecf3cee2b3c7a5ac9e1234506f6f5f3ce1230e123
    ```

3. Navigate to the settings of your repository on Github, under "Settings/Webhooks" press the "Add webhook" button. Fill in the webhook url and your `bootstrap_flux_github_webhook_token` secret and save.

### 🤖 Renovate

[Renovate](https://www.mend.io/renovate) is a tool that automates dependency management. It is designed to scan your repository around the clock and open PRs for out-of-date dependencies it finds. Common dependencies it can discover are Helm charts, container images, GitHub Actions, Ansible roles... even Flux itself! Merging a PR will cause Flux to apply the update to your cluster.

To enable Renovate, click the 'Configure' button over at their [Github app page](https://github.com/apps/renovate) and select your repository. Renovate creates a "Dependency Dashboard" as an issue in your repository, giving an overview of the status of all updates. The dashboard has interactive checkboxes that let you do things like advance scheduling or reattempt update PRs you closed without merging.

The base Renovate configuration in your repository can be viewed at [.github/renovate.json5](https://github.com/onedr0p/flux-cluster-template/blob/main/.github/renovate.json5). By default it is scheduled to be active with PRs every weekend, but you can [change the schedule to anything you want](https://docs.renovatebot.com/presets-schedule), or remove it if you want Renovate to open PRs right away.

## 🐛 Debugging

Below is a general guide on trying to debug an issue with an resource or application. For example, if a workload/resource is not showing up or a pod has started but in a `CrashLoopBackOff` or `Pending` state.

1. Start by checking all Flux Kustomizations & Git Repository & OCI Repository and verify they are healthy.

    ```sh
    flux get sources oci -A
    flux get sources git -A
    flux get ks -A
    ```

2. Then check all the Flux Helm Releases and verify they are healthy.

    ```sh
    flux get hr -A
    ```

3. Then check the if the pod is present.

    ```sh
    kubectl -n <namespace> get pods -o wide
    ```

4. Then check the logs of the pod if its there.

    ```sh
    kubectl -n <namespace> logs <pod-name> -f
    # or
    stern -n <namespace> <fuzzy-name>
    ```

5. If a resource exists try to describe it to see what problems it might have.

    ```sh
    kubectl -n <namespace> describe <resource> <name>
    ```

6. Check the namespace events

    ```sh
    kubectl -n <namespace> get events --sort-by='.metadata.creationTimestamp'
    ```

Resolving problems that you have could take some tweaking of your YAML manifests in order to get things working, other times it could be a external factor like permissions on NFS. If you are unable to figure out your problem see the help section below.

### Updates

Use `git cherry-pick` to keep up-to-date

```sh
git remote add upstream https://github.com/onedr0p/flux-cluster-template.git
git fetch --all
# for individual commits
git cherry-pick <commitid>
# brute force
git merge upstream/main --allow-unrelated-histories
# remove upstream after
git remote rm upstream
```

### Authenticate Flux over SSH

Authenticating Flux to your git repository has a couple benefits like using a private git repository and/or using the Flux [Image Automation Controllers](https://fluxcd.io/docs/components/image/).

By default this template only works on a public Github repository, it is advised to keep your repository public.

The benefits of a public repository include:

- Debugging or asking for help, you can provide a link to a resource you are having issues with.
- Adding a topic to your repository of `k8s-at-home` to be included in the [k8s-at-home-search](https://nanne.dev/k8s-at-home-search/). This search helps people discover different configurations of Helm charts across others Flux based repositories.

<!-- markdownlint-disable MD033 -->
<details>
  <summary>Expand to read guide on adding Flux SSH authentication</summary>
<!-- markdownlint-enable MD033 -->

1. Generate new SSH key:

    ```sh
    ssh-keygen -t ecdsa -b 521 -C "github-deploy-key" -f ./kubernetes/bootstrap/github-deploy.key -q -P ""
    ```

2. Paste public key in the deploy keys section of your repository settings

3. Create sops secret in `./kubernetes/bootstrap/github-deploy-key.sops.yaml` with the contents of:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: github-deploy-key
     namespace: flux-system
   stringData:
     # 3a. Contents of github-deploy-key
     identity: |
       -----BEGIN OPENSSH ... -----
           ...
       -----END OPENSSH ... -----
     # 3b. Output of curl --silent https://api.github.com/meta | jq --raw-output '"github.com "+.ssh_keys[]'
     known_hosts: |
       github.com ssh-ed25519 ...
       github.com ecdsa-sha2-nistp256 ...
       github.com ssh-rsa ...
   ```

4. Encrypt secret:

    ```sh
    sops --encrypt --in-place ./kubernetes/bootstrap/github-deploy-key.sops.yaml
    ```

5. Apply secret to cluster:

    ```sh
    sops --decrypt ./kubernetes/bootstrap/github-deploy-key.sops.yaml | kubectl apply -f -
    ```

6. Update `./kubernetes/flux/config/cluster.yaml`:

    ```yaml
    apiVersion: source.toolkit.fluxcd.io/v1beta2
    kind: GitRepository
    metadata:
      name: home-kubernetes
      namespace: flux-system
    spec:
      interval: 10m
      # 6a: Change this to your user and repo names
      url: ssh://git@github.com/$user/$repo
      ref:
        branch: main
      secretRef:
        name: github-deploy-key
    ```

7. Commit and push changes
8. Force flux to reconcile your changes

    ```sh
    flux reconcile -n flux-system kustomization cluster --with-source
    ```

9. Verify git repository is now using SSH:

    ```sh
    flux get sources git -A
    ```

10. Optionally set your repository to Private in your repository settings.

<!-- markdownlint-disable MD033 -->
</details>
<!-- markdownlint-enable MD033 -->

## 🤝 Thanks

This would not be possible without onedr0p and the k8s-at-home team
