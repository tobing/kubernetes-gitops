# Kubernetes GitOps with K3s and ArgoCD


## Prerequisite
- 1 VM for master node **k3s-master01** (2 vCPU, 8GB RAM, 30GB Storage)
- 1 VM for worker node **k3s-worker01** (2 vCPU, 4GB RAM, 30GB Storage)
- Both VM running Ubuntu Server (tested on v26.04)
##


1. In Proxmox host, enable forwarding
   
    ```sudo sysctl -w net.ipv4.ip_forward=1```

    **PERMANENT** 

    ``` echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/99-ip-forward.conf``` 


2. Create lxc container with OPTION like this image, set hostname "k3s-master01"
    <img width="696" height="469" alt="image" src="https://github.com/user-attachments/assets/c49996b7-092a-4ed2-a2c7-9ca729d13da1" />



3. Create another lxc container same as above but set RAM 2GB and hostname "k3s-worker01"
4. In Proxmox host, for each container modify value of ```/etc/pve/lxc/<lxc_container_id>.conf```
   
   ```nano /etc/pve/lxc/<lxc_container_id>.conf``` and append these configs

   ```
   lxc.apparmor.profile: unconfined
   lxc.cgroup.devices.allow: a
   lxc.cap.drop:
   lxc.mount.auto: "proc:rw sys:rw"
   ```

5. For each container, modify ```/etc/rc.local```.

    Check if file exist ```ls –al /etc/rc.local```  

    If not exist ```nano /etc/rc.local``` and put these

    ```
   #!/bin/sh -e
   if [ ! -e /dev/kmsg ]; then
       ln -s /dev/console /dev/kmsg
   fi
   mount --make-rshared /
    ```
 
    Change permission to executable  


    ```chmod +x /etc/rc.local ```

  
    Run apt update and install curl then reboot the containers
   
    ```apt update && apt upgrade –y && apt install curl –y && reboot```

6. On ```k3s-master01```
   
    Install K3s master/control plane
   
    ```curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable servicelb" sh -s - --write-kubeconfig-mode 644```

    Get K3s token for worker node installation

    ```cat /var/lib/rancher/k3s/server/node-token```
   
7. On ```k3s-worker01```
    Install K3s worker/agent

    ```curl -sfL https://get.k3s.io | K3S_URL=https://<mymasternode>:6443 K3S_TOKEN=<mymasternodetoken> sh -```

   ```<mymasternode>``` is k3s-master01 IP Address or hostname

   ```<mymasternodetoken>``` is the token from step 6


1. On ```k3s-master01```
   
    Install K3s master/control plane<br>
    ```curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable servicelb" sh -s - --write-kubeconfig-mode 644```

    Get K3s token for worker node installation<br>
    ```cat /var/lib/rancher/k3s/server/node-token```
   
2. On ```k3s-worker01```

   Install K3s worker/agent<br>
    ```curl -sfL https://get.k3s.io | K3S_URL=https://<mymasternode>:6443 K3S_TOKEN=<mymasternodetoken> sh -```
   
   ```<mymasternode>``` is k3s-master01 IP Address or hostname<br>
   ```<mymasternodetoken>``` is the token from step 1
   
3. Install ArgoCD to the cluster

     ```
     kubectl create namespace argocd     
     kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
     ```

     Get secret from ArgoCD
     ```
     kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
     ```

     Port Forwarding for tunneling
     ```
     kubectl port-forward svc/argocd-server -n argocd 8081:443
     ```

4. From you computer

      Tunneling to ```k3s-master01_IP_ADDRESS``` with existing ```USERNAME```       
      ```
      ssh -L 8081:127.0.0.1:8081 USERNAME@k3s-master01_IP_ADDRESS
      ```
      Access ArgoCD Web UI from web browser with user ```admin``` and secret from step 3
      ```
      https://127.0.0.1:8081
   
      ```

> [!NOTE]
> **Prepare your own repo as source of ArgoCD to deploy apps, you can clone this repo to your github account**

<details>
  <summary>For Private Repo, You need to generate key</summary>
  

   
5. Create key for git repo

   ```
   ssh-keygen -t ed25519 -f ~/.ssh/argocd-github -C "argocd-kubernetes-gitops"
   ```

   Copy content of public key
   
   ```
   cat ~/.ssh/argocd-github.pub
   ```
6. Add it to GitHub

   In your repo example ```tobing/kubernetes-gitops``` repository:<br/>
   Go to ```Settings → Deploy keys → Add deploy key```<br/>
   Set:<br/>
   **Title**: ArgoCD<br/>
   **Key**: paste the contents of argocd-github.pub<br/>
   **Allow write access**: OFF<br/>
   We only want ArgoCD to **read** Git.<br/>
   
   
7. From the machine where ```~/.ssh/argocd-github``` exists:

   ```
   kubectl create secret generic repo-kubernetes-gitops \
   -n argocd \
   --from-literal=type=git \
   --from-literal=url=git@github.com:tobing/kubernetes-gitops.git \
   --from-file=sshPrivateKey=$HOME/.ssh/argocd-github
   ```
  
8. Then label it so ArgoCD recognizes it as a repository credential:

      ```
      kubectl label secret repo-kubernetes-gitops \
      -n argocd \
      argocd.argoproj.io/secret-type=repository
      ```
  
</details>
  
9. Create a temporary file [`root-application.yaml`](root-application.yaml) then run ```kubectl apply -f root-application.yaml```<br/>
⚠️ This is for ArgoCD initialization. <br> The content of this file is the structure of the git repo. **CHECK CAREFULLY!!**


10. dsd
