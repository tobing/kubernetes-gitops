## Kubernetes GitOps with K3s, ArgoCD and Proxmox VM

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

### Environments
- 1 VM for master node **k3s-master01** <br>
  (2 vCPU, +8GB RAM, +50GB Storage) - IP Addr 192.168.31.4 (set according to your local subnet)
- 1 VM for worker node **k3s-worker01** <br>
  (2 vCPU, +8GB RAM, +50GB Storage) - IP Addr 192.168.31.5 (set according to your local subnet)
- Both running Ubuntu Server (tested on v26.04)
##

1. In ```k3s-master01```
   
    Install K3s master/control plane without servicelb. We will replace it with Metal LB<br>
    ```
   curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable servicelb" sh -s - --write-kubeconfig-mode 644
    ```

    Verify with
    ```
   systemctl status k3s
    ```  

    Get K3s token for worker node installation<br>
    ```
   cat /var/lib/rancher/k3s/server/node-token
    ```
   
2. In ```k3s-worker01```

   Install K3s worker/agent<br>
    ```
   curl -sfL https://get.k3s.io | K3S_URL=https://<mymasternode>:6443 K3S_TOKEN=<mymasternodetoken> sh -
    ```
   
   ```<mymasternode>``` is k3s-master01 IP Address or hostname<br>
   ```<mymasternodetoken>``` is the token from step 1
   
3. From ```k3s-master01``` install ArgoCD to the cluster

    Check status both nodes
    ```
    kubectl get nodes
    ```

   Create namespace and install ArgoCD
     ```
     kubectl create namespace argocd     
     kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
     ```

     Wait few minutes to complete ArgoCD deployment, check with ```kubectl get all -n argocd```

     Get ArgoCD secret
     ```
     kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
     ```

     Port Forwarding for tunneling
     ```
     kubectl port-forward svc/argocd-server -n argocd 8081:443
     ```

5. From you computer

      Tunneling to ```k3s-master01_IP_ADDRESS``` with existing ```USERNAME```       
      ```
      ssh -L 8081:127.0.0.1:8081 USERNAME@k3s-master01_IP_ADDRESS
      ```
      Access ArgoCD Web UI from web browser with user ```admin``` and secret from step 3
      ```
      https://127.0.0.1:8081
   
      ```
> [!NOTE]
> **Prepare your own repo as source of ArgoCD to deploy apps, you can clone this repo to your github account.** <br>
> **If you don't want to deploy Longhorn, remove longhorn from infrastructure directory**
<details>
  <summary><h4>For Private Repo, you need to generate key for ArgoCD so it can access your repo</h4></summary>
   
5. In ```k3s-master01``` create key for git repo. **Skip passphrase** (leave it empty)

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
   
   
7. From ```k3s-master01``` create secret:

     ```
     kubectl create secret generic repo-kubernetes-gitops \
     -n argocd \
     --from-literal=type=git \
     --from-literal=url=git@github.com:tobing/kubernetes-gitops.git \
     --from-file=sshPrivateKey=$HOME/.ssh/argocd-github
     ```
     Modify```git@github.com:tobing/kubernetes-gitops.git``` to your git repo
  
8. Then label it so ArgoCD recognizes it as a repository credential:

      ```
      kubectl label secret repo-kubernetes-gitops \
      -n argocd \
      argocd.argoproj.io/secret-type=repository
      ```
  
</details>
  
9. Check [`infrastructure-config/metallb/ipaddresspool.yaml`](infrastructure-config/metallb/ipaddresspool.yaml) for your Metal LB IP Address pool.<br> I am using ```192.168.31.240-192.168.31.250```. Set them based on your local subnet.
10. Create a temporary file [`root-application.yaml`](root-application.yaml) ⚠️ This file for ArgoCD initialization.<br/>
    Modify ```repoURL``` according to your git repo. $\color{red}{\text{CHECK CAREFULLY}}$ <br/>
    Run ```kubectl apply -f root-application.yaml```


11. Try to modify metallb version ```targetRevision: 0.16.1``` in [`infrastructure/metallb/application.yaml`](infrastructure/metallb/application.yaml) to something else like ```0.16.0```. <br>
    After git push, ArgoCD will syncing.

> [!NOTE]
> **Longhorn - Distributed Block Storage System for Kubernetes**
>

<details>

12. Install iSCSI in both ```k3s-master01``` and ```k3s-worker01```

    ```
    apt update
    apt install -y open-iscsi
    ```

    Enable the services
    
    ```
    systemctl enable --now iscsid
    systemctl enable --now open-iscsi
    ```

    Check
    ```
    systemctl status iscsid.socket --no-pager
    systemctl status iscsid --no-pager
    ```

13. Because we use 2 nodes only, but Longhorn default replica is 3 
    ```
    kubectl -n longhorn-system get settings.longhorn.io default-replica-count -o yaml
    ```
    ```
    value: '{"v1":"3","v2":"3"}'
    ```

    Change to 2
    ```
    kubectl -n longhorn-system patch settings.longhorn.io default-replica-count \
    --type=merge \
    -p '{"value":"{\"v1\":\"2\",\"v2\":\"2\"}"}'
    ```

    Verify
    ```
    kubectl -n longhorn-system get settings.longhorn.io default-replica-count -o yaml
    ```
    

14. Create a test Persistant Volume Claim (PVC)

    ```
    kubectl create -f - <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: longhorn-test
    spec:
      storageClassName: longhorn
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 2Gi
    EOF
    ```

    Check
    ```
    kubectl get pvc longhorn-test
    ```

    Result
    ```
    STATUS   Bound
    ```

15. Create a test pods
    ```
    kubectl create -f - <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: longhorn-test
    spec:
      containers:
        - name: test
          image: busybox
          command: ["sh", "-c", "echo 'Longhorn works!' > /data/test.txt && sleep 3600"]
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: longhorn-test
    EOF
    ```

    Check if running
    ```
    kubectl get pod longhorn-test
    ```

    If running
    ```
    kubectl exec longhorn-test -- cat /data/test.txt
    ```

    Expected result
    ```
    Longhorn works!
    ```
    
16. Access Longhorn Web UI
    
    From ```k3s-master01```
    ```
    kubectl port-forward -n longhorn-system svc/longhorn-frontend 8090:80
    ```

    From you computer tunneling to ```k3s-master01_IP_ADDRESS``` with existing ```USERNAME```       
    ```
    ssh -L 8090:127.0.0.1:8090 USERNAME@k3s-master01_IP_ADDRESS
    ```
    Access ArgoCD Web UI from web browser
    ```
    http://127.0.0.1:8090

</details>
