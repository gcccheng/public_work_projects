

# GitHub Actions Runner Controller (ARC) on Kubernetes

This document consolidates the design, setup, and advanced usage of GitHub Actions Runner Controller (ARC) in Kubernetes. It includes deployment at **repository level** and **organization level**, along with advanced configurations such as Docker-in-Docker and Kubernetes modes.

---

## 1. Kubernetes Setup (Foundation)

ARC requires a working Kubernetes cluster. Below are the setup steps.

### Prerequisites

Ensure firewall allows access to:

* quay.io
* actions-runner-controller.github.io
* pkgs.k8s.io
* githubusercontent.com/projectcalico
* ghcr.io/actions/actions-runner-controller-charts
* get.helm.sh
* api.github.com

### Node Configuration

On all nodes:

1. Configure hostnames (`/etc/hosts`) to include control plane and worker nodes.
2. Load kernel modules:

   ```bash
   modprobe overlay
   modprobe br_netfilter
   ```
3. Sysctl settings:

   ```bash
   net.ipv4.ip_forward=1
   net.bridge.bridge-nf-call-ip6tables=1
   net.bridge.bridge-nf-call-iptables=1
   ```
4. Disable swap (`swapoff -a`).
5. Add Kubernetes and Docker repos.
6. Install Kubernetes components (`kubelet`, `kubeadm`, `kubectl`, `containerd.io`).
7. Configure containerd with `SystemdCgroup=true`.
8. Configure firewall:

   * Control plane ports: 6443, 2379, 2380, 10250–10252, 5473, 53
   * Worker node: allow control-plane IP access to 6443.

### Control Plane

```bash
kubeadm init --pod-network-cidr=10.10.0.0/16
```

* Copy admin config to `~/.kube/config`.
* Install CNI (Calico).

### Worker Nodes

Join using token from control plane:

```bash
kubeadm join --token <TOKEN> --discovery-token-ca-cert-hash <HASH>
```

---

## 2. ARC Design Overview

ARC is a Kubernetes operator that orchestrates and scales GitHub Actions runners.

### Benefits

* **Stability**: isolated environments
* **Scalability**: auto-scale runners
* **Efficiency**: resource control, faster execution
* **Control**: centralized runner management

### Control Plane

* API Server (6443/tcp)
* Scheduler (6443/tcp)
* Controller Manager (6443/tcp)
* etcd (2379, 2380/tcp)

### Worker Nodes

* kubelet (10250/tcp)
* kube-proxy (10256/tcp)


## 3. ARC Setup at Repository Level (with PAT)

### Steps

1. Log in to GitHub registry:

   ```bash
   echo $PAT | docker login ghcr.io -u USERNAME --password-stdin
   ```
2. Install ARC operator:

   ```bash
   helm install arc --namespace arc-systems --create-namespace \
     oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller --version 0.93
   ```
3. Create PAT with required scopes (`repo`, `admin:org`).
4. Configure runner scale set:

   ```bash
   helm install arc-runner-set --namespace arc-runners --create-namespace \
     --set githubConfigUrl="https://github.com/<org>/<repo>" \
     --set githubConfigSecret.github_token="$PAT" \
     oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
   ```
5. Create secret for PAT:

   ```bash
   kubectl create secret generic pre-defined-secret -n arc-runners \
     --from-literal=github_token=$PAT
   ```
6. Verify installation with `helm list -A` and check pods.

### Customization

Modify runner set values to use:

* **Custom runner image**
* **HostPath volumes** (e.g., `/mydata`)

Update with:

```bash
helm upgrade arc-runner-set oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  -f values.yaml -n arc-runners
```

---
Sample values.yml file

```yaml

githubConfigUrl: https://github.com/your-org/your-repo
githubConfigSecret: your-secret
template:
  spec:
    volumes:
      - name: data-volume
        hostPath:
          path: /mydata # path on the host filesystem
          type: Directory
    containers:
      - name: runner
        image: your-internal-registry/arc-image:latest
          # to use default image: ghcr.io/actions/actions-runner:latest
        command: ["/home/runner/run.sh"]
        volumeMounts:
          - name: data-volume
            mountPath: /mydata # mount path inside the container
    imagePullSecrets: #if it is required a key to pull the image, then create a secret based on the key
      - name: registry-secret # Create the secret in the same namespace.
```


## 4. ARC Setup at Organization Level (with GitHub App)

### Steps

1. Create a GitHub App:

   * Homepage URL: `https://github.com/actions/actions-runner-controller`
   * Permissions:

     * Repositories: Administration (RW), Metadata (RO)
     * Organization: Self-hosted runners (RW)
   * Generate private key (.pem), note App ID & Installation ID.

2. Create Kubernetes secret:

   ```bash
   kubectl create secret generic github-app-secret -n arc-org \
     --from-literal=github_app_id=<ID> \
     --from-literal=github_app_installation_id=<ID> \
     --from-file=github_app_private_key=private-key.pem
   ```

3. Install ARC runner set:

   ```bash
   helm install org-arc-runner-set --namespace arc-org \
     --set githubConfigUrl="https://github.com/<org>" \
     --set githubConfigSecret.existingSecret=github-app-secret \
     oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
   ```

4. Create registry secret to pull private images.

5. Customize values file to specify runner groups, hostPath mounts, and custom images.

6. Upgrade with:

   ```bash
   helm upgrade org-arc-runner-set oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
     -f org_arc_set_values.yaml -n arc-org
   ```

---

## 5. ARC Advanced Configurations

ARC supports two advanced modes:

### A. Kubernetes Mode

Concept

The runner container stays close to upstream (ghcr.io/actions/actions-runner).

Each job launches a separate job container defined in the workflow’s container: block.

You keep using container: in workflows and pin your own dependency image per job.

### B. Docker-in-Docker (DinD) Mode

Concept

A single pod runs three containers:

Runner (ARC runner)

DinD (docker:dind providing /var/run/docker.sock)

Job container (your container: image, launched by the runner using the DinD socket)

All three share a hostPath mount (e.g., /mydata) so the job container can write to a real directory on the node.

#### Challenges & Solutions

1. **File Persistence**

   * Mount hostPath shared volume.
2. **Checkout Errors** (Node.js missing)

   * Use init container to preload tools.
3. **Permission Issues**

   * Solution 1: Run as `--user root`
   * Solution 2: Match UID between runner & job container.
4. **Image Pull Performance**

   * Use persistent Docker cache in DinD container.

---

Sample values.yml file for DinD mode

Sample values.yml file

```yaml
# dind_mode_values.yaml
githubConfigUrl: "https://github.com/<ORG>"        
githubConfigSecret: github-app-secret                 
runnerGroup: "group-dev"                          
minRunners: 1
maxRunners: 4

template:
  spec:
    initContainers:
      # Ensure runner runtime externals (e.g., Node.js) are available for some actions (checkout)
      - name: init-runner
        image: ghcr.io/actions/actions-runner:latest
        imagePullPolicy: IfNotPresent
        command: ["sh", "-c", "cp -ar /home/runner/externals/* /externals/"]
        volumeMounts:
          - name: externals
            mountPath: /externals
          - name: host-time
            mountPath: /etc/localtime
            readOnly: true

    containers:
      # 1) Runner container
      - name: runner
        image: ghcr.io/actions/actions-runner:latest
        command: ["/home/runner/run.sh"]
        env:
          - name: DOCKER_HOST
            value: unix:///var/run/docker.sock
        volumeMounts:
          - name: work
            mountPath: /home/runner/_work
          - name: dind-sock
            mountPath: /var/run
          - name: shared-out # this is the volume defined in the later part of this file, this volume is then mounted in to the dind container under /mydata
            mountPath: /mydata 
          - name: externals
            mountPath: /home/runner/externals
          - name: host-time
            mountPath: /etc/localtime
            readOnly: true

      # 2) DinD container
      - name: dind
        image: docker:dind
        args:
          - dockerd
          - --host=unix:///var/run/docker.sock
          - --group=$(DOCKER_GROUP_GID)
        env:
          - name: DOCKER_GROUP_GID
            value: "123"
        securityContext:
          privileged: true
        volumeMounts:
          - name: work
            mountPath: /home/runner/_work
          - name: dind-sock
            mountPath: /var/run
          - name: shared-out # this is the volume defined in the later part of this file, this volume is then mounted in to the dind container under /mydata
            mountPath: /mydata
          - name: externals
            mountPath: /home/runner/externals
          - name: docker-cache                   # Persistent image cache
            mountPath: /var/lib/docker
          - name: host-time
            mountPath: /etc/localtime
            readOnly: true

    volumes:
      - name: work
        emptyDir: {}
      - name: dind-sock
        emptyDir: {}
      - name: shared-out
        hostPath:
          path: /mydata # this is the directory on the local work node file system, this directory will act as a volume named shared-out
          type: DirectoryOrCreate
      - name: externals
        emptyDir: {}
      - name: docker-cache
        hostPath:
          path: /var/lib/docker-cache
          type: DirectoryOrCreate
      - name: host-time
        hostPath:
          path: /etc/localtime
          type: File

```
Sample workflow file:

```yaml
name: test ARC dind mode
on:
  workflow_dispatch:
permissions:
  contents: read

jobs:
  # Job 1: To inspect runner container filesystem
  inspect-runner:
    runs-on: org-arc-runner-set
    steps:
      - name: Inspect runner container file system
        run: |
          echo "show id"
          id
          echo "show runner container file system"
          ls -l /
          echo "show /mydata" # the /mydata folder is mounted from the shared-out volume
          ls -l /mydata 
                                                                                                                           1,7           Top
# Job 2: To run inside a container
  Container-job:
    runs-on: org-arc-runner-set

    # Job container block
    container:
      image: your-registry/image:latest
      credentials:
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
      options: --user root # you can comment this line if you fix uid id issue
      volumes:
        - /mydata:/data # this is to mount the /mydata in the runner container into the job container /data
steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Show container file system
        run: |
          echo "show id"
          id
          echo "show partition"
          df -h
          echo "show mount"
          mount
          echo "[$(date)] this is a test from docker in docker" >> /data/${{ github.repository }}/testdind.txt # save a test file into the /data folder, the file will be also saved into worker node under local file system /mydata
        
```

Why the init-runner container?


Some actions (notably actions/checkout) need Node.js and other “externals” present at expected paths. The init container copies /home/runner/externals/* into a shared volume that both runner and DinD mount, ensuring the runtime pieces exist even though the runner image stays close to upstream.

Mount Chian

Host node /mydata
  └──(hostPath)──► Runner container /mydata
        └──(container volume)──► Job container /data

So the /mydata on local work node file system is mounted into the runner container /mydata which is then mounted into the job container /data. Workflow jobs can save file directly into the /data in the container, which will be saved on local work node file system /mydata folder through the mounting chain.

Why options: --user root?

The ARC runner’s workspace mounts (e.g., /_work, /__w) are owned by the runner user inside the runner container (often UID 1001).

Your job container image may use a different UID (e.g., ghrunner with UID 1002), causing EACCES when actions write to /_temp/_runner_file_commands or similar.

Running as root avoids brittle UID gymnastics and is acceptable here because the job container is already constrained inside the pod and DinD boundary.

Alternative (more strict): Modify your job image to use UID 1001:

# In your runner-environment Dockerfile
RUN groupadd -g 1001 ghrunner \
    && useradd -m -u 1001 -g 1001 ghrunner \
    && chown -R ghrunner:ghrunner /opt
USER ghrunner

DinD image pull performance

Without caching, every job’s first container pull happens inside DinD, missing the node’s containerd cache.

The docker-cache hostPath mount (/var/lib/docker-cache) makes pulls persistent across DinD restarts → big time saver.

Do not use :latest. Pin a version tag (runner-environment:YYYY.MM.DD), or implement a periodic cache refresh if you must track “latest”.

Security cautions (DinD)

privileged: true is required for DinD. Limit which namespaces/nodes run these pods.

Keep secrets out of container logs.

Prefer GitHub App auth at org level; rotate keys after any App config change (you must re-create the K8s secret with the new .pem).


Storage Design Notes

Simple & local: use hostPath → fast, node-local. Good when runner sets are pinned to specific worker nodes.

Shared: back hostPath with a mounted SMB/NFS directory on workers, then mount into the pods (via hostPath). This gives you one source of truth reachable from any runner pod scheduled to that node.

Ephemeral PVCs (K8s mode): fine for temporary workspaces; not for cross-job persistence.


## 6. Summary

This is really comlext setup, not recommended.

1. **Kubernetes cluster** must be set up first.
2. **ARC can be deployed** at:

   * Repository level (via PAT)
   * Organization level (via GitHub App + Runner Groups)
3. **Advanced setups** (Kubernetes mode, DinD mode) provide flexibility but require volume persistence, permission alignment, and caching.
4. **Storage design**: use hostPath or Samaba for persistent storage to save artifacts.
