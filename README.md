## Kubernetes GitOps with K3s, ArgoCD and LXC container

```
kubernetes-gitops/
├── root-application.yaml
│
├── infrastructure/
│   ├── metallb/
│   │   └── application.yaml
│   └── longhorn/
│       └── application.yaml
│
└── infrastructure-config/
    └── metallb/
        ├── ipaddresspool.yaml
        └── l2advertisement.yaml
```

### Prerequisite
- 1 LXC container for master node **k3s-master01** (2 vCPU, 4GB RAM, 20GB Storage)
- 1 LXC container for worker node **k3s-worker01** (2 vCPU, 2GB RAM, 20GB Storage)
- Both containers running Ubuntu Server (tested on v26.04)
  
🎈 if using VM instead of lxc container, go directly to step 6.
##


1. In Proxmox host
   
   Enable forwarding<br>
   ```sysctl -w net.ipv4.ip_forward=1```

    **PERMANENT** 

    ``` echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/99-ip-forward.conf```

   Check overlay and br_netfilter modules 
   ```
   lsmod | grep overlay && grep CONFIG_BRIDGE_NETFILTER /boot/config-$(uname -r)
   ```
   Will return almost like this
   ```
   overlay               xxxxxx  xx
   CONFIG_BRIDGE_NETFILTER=y
   ```

2. Create lxc container with OPTION like this image, set hostname to "**k3s-master01**"
    <img width="481" height="445" alt="image" src="https://github.com/user-attachments/assets/200e7b54-35fd-49dd-9521-efe4d2176183" />


3. Create another lxc container same as above but set RAM 2GB and hostname "**k3s-worker01**"
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
   
    ```apt update && apt upgrade -y && apt install curl -y && reboot```


6. In ```k3s-master01```
   
    Install K3s master/control plane<br>
    ```curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable servicelb" sh -s - --write-kubeconfig-mode 644```

    Verify with
    ```
   systemctl status k3s
    ```
    It will show k3s service active but running k3s inside lxc container will show these errors
   ```
   ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=1/FAILURE)
   ExecStartPre=/sbin/modprobe overlay (code=exited, status=1/FAILURE)
   ```
   As expected, lxc container using host's kernel so cannot run modprobe to load kernel modules<br>
   (Step 1: check Proxmox host loaded the modules) 

    Get K3s token for worker node installation<br>
    ```cat /var/lib/rancher/k3s/server/node-token```
   
7. In ```k3s-worker01```

   Install K3s worker/agent<br>
    ```curl -sfL https://get.k3s.io | K3S_URL=https://<mymasternode>:6443 K3S_TOKEN=<mymasternodetoken> sh -```
   
   ```<mymasternode>``` is k3s-master01 IP Address or hostname<br>
   ```<mymasternodetoken>``` is the token from step 1
   
8. Install ArgoCD to the cluster

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

9. From you computer

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
  <summary><h3>For Private Repo, You need to generate key</h3></summary>
  

   
10. In ```k3s-master01``` create key for git repo

     ```
     ssh-keygen -t ed25519 -f ~/.ssh/argocd-github -C "argocd-kubernetes-gitops"
     ```
  
     Copy content of public key
     
     ```
     cat ~/.ssh/argocd-github.pub
     ```

11. Add it to GitHub

     In your repo example ```tobing/kubernetes-gitops``` repository:<br/>
     Go to ```Settings → Deploy keys → Add deploy key```<br/>
     Set:<br/>
     **Title**: ArgoCD<br/>
     **Key**: paste the contents of argocd-github.pub<br/>
     **Allow write access**: OFF<br/>
     We only want ArgoCD to **read** Git.<br/>
   
   
12. From ```k3s-master01`` create secret:

     ```
     kubectl create secret generic repo-kubernetes-gitops \
     -n argocd \
     --from-literal=type=git \
     --from-literal=url=git@github.com:tobing/kubernetes-gitops.git \
     --from-file=sshPrivateKey=$HOME/.ssh/argocd-github
     ```
  
13. Then label it so ArgoCD recognizes it as a repository credential:

      ```
      kubectl label secret repo-kubernetes-gitops \
      -n argocd \
      argocd.argoproj.io/secret-type=repository
      ```
  
</details>
  
14. Create a temporary file [`root-application.yaml`](root-application.yaml) then run ```kubectl apply -f root-application.yaml```<br/>
⚠️ This is for ArgoCD initialization. <br> The content of this file is the structure of the git repo. **CHECK CAREFULLY!!**


15. Check [`infrastructure-config/metallb/ipaddresspool.yaml`](infrastructure-config/metallb/ipaddresspool.yaml) for your Metal LB IP Address pool.<br> Set them based on your network.
16. Try to modify metallb version ```targetRevision: 0.16.1``` in [`infrastructure/metallb/application.yaml`](infrastructure/metallb/application.yaml) to something else like ```0.16.0```. <br>
    After git push, ArgoCD will syncing.
