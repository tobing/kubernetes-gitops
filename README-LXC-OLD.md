## Kubernetes GitOps with K3s, ArgoCD and LXC container

```
kubernetes-gitops/
├── root-application.yaml
│
├── infrastructure/
│   └── metallb/
│       └── application.yaml
│
└── infrastructure-config/
    └── metallb/
        ├── ipaddresspool.yaml
        └── l2advertisement.yaml
```

### Environments
- 1 LXC container for master node **k3s-master01** <br>
  (2 vCPU, 4GB, no swap, 25GB Storage) - IP Addr 192.168.31.4 (set same as your local subnet)
- 1 LXC container for worker node **k3s-worker01** <br>
  (2 vCPU, 2GB, no swap, 25GB Storage) - IP Addr 192.168.31.5 (set same as your local subnet)
- Both containers running Debin/Ubuntu Server (tested Debian 13 & Ubuntu 26.04)
  
🎈 $\color{red}{\text{LXC Container encountered error with iscsi, Loghorn cannot run on LXC container.}}$ <br> 🎈 $\color{red}{\text{!! Remove longhorn from infrastructure directory !!}}$

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
   
    Install K3s master/control plane without servicelb. We will replace it with Metal LB<br>
    ```
   curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable servicelb" sh -s - --write-kubeconfig-mode 644
    ```

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
    ```
   cat /var/lib/rancher/k3s/server/node-token
    ```
   
8. In ```k3s-worker01```

   Install K3s worker/agent<br>
    ```
   curl -sfL https://get.k3s.io | K3S_URL=https://<mymasternode>:6443 K3S_TOKEN=<mymasternodetoken> sh -
    ```
   
   ```<mymasternode>``` is k3s-master01 IP Address or hostname<br>
   ```<mymasternodetoken>``` is the token from step 1
   
10. From ```k3s-master01``` install ArgoCD to the cluster

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

11. From you computer

      Tunneling to ```k3s-master01_IP_ADDRESS``` with existing ```USERNAME```       
      ```
      ssh -L 8081:127.0.0.1:8081 USERNAME@k3s-master01_IP_ADDRESS
      ```
      Access ArgoCD Web UI from web browser with user ```admin``` and secret from step 8
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
   
   
12. From ```k3s-master01``` create secret:

     ```
     kubectl create secret generic repo-kubernetes-gitops \
     -n argocd \
     --from-literal=type=git \
     --from-literal=url=git@github.com:tobing/kubernetes-gitops.git \
     --from-file=sshPrivateKey=$HOME/.ssh/argocd-github
     ```
     Modify```git@github.com:tobing/kubernetes-gitops.git``` to your git repo
  
13. Then label it so ArgoCD recognizes it as a repository credential:

      ```
      kubectl label secret repo-kubernetes-gitops \
      -n argocd \
      argocd.argoproj.io/secret-type=repository
      ```
  
</details>
  
14. Create a temporary file [`root-application.yaml`](root-application.yaml) ⚠️ This file for ArgoCD initialization.<br/>
    Modify ```repoURL``` according to your git repo. $\color{red}{\text{CHECK CAREFULLY}}$ <br/>
    Run ```kubectl apply -f root-application.yaml```

15. Check [`infrastructure-config/metallb/ipaddresspool.yaml`](infrastructure-config/metallb/ipaddresspool.yaml) for your Metal LB IP Address pool.<br> Set them based on your local subnet.
16. Try to modify metallb version ```targetRevision: 0.16.1``` in [`infrastructure/metallb/application.yaml`](infrastructure/metallb/application.yaml) to something else like ```0.16.0```. <br>
    After git push, ArgoCD will syncing.

    🎉 $\color{red}{\text{You have implemented GitOps by using your git repo as source of truth}}$ 🎉

<br><br>

> [!NOTE]
> **Longhorn - Distributed Block Storage (LXC Container issue with iscsi)**
>

<details>

17. In Proxmox host enable iSCSI module

    ```
    modprobe iscsi_tcp
    
    #check
    lsmod | grep iscsi
    ```

    **PERMANENT**
    
    ```
    echo iscsi_tcp | tee /etc/modules-load.d/iscsi.conf

    #check
    cat /etc/modules-load.d/iscsi.conf
    ```  
18. Install iSCSI in both ```k3s-master01``` and ```k3s-worker01```

    ```
    apt update
    apt install -y open-iscsi
    ```

    Enable the services
    
    ```
    sudo systemctl enable --now iscsid
    sudo systemctl enable --now open-iscsi
    ```

    Check
    ```
    systemctl status iscsid.socket --no-pager
    systemctl status iscsid --no-pager
    ```

    $\color{red}{\text{Services failed in LXC Container}}$
    

</details>
