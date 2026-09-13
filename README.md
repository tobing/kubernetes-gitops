# kubernetes-gitops-lxc

1. On ```k3s-master01```
   
    Install K3s master/control plane
   
    ```curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable servicelb" sh -s - --write-kubeconfig-mode 644```

    Get K3s token for worker node installation

    ```cat /var/lib/rancher/k3s/server/node-token```
   
2. On ```k3s-worker01```
    Install K3s worker/agent

    ```curl -sfL https://get.k3s.io | K3S_URL=https://<mymasternode>:6443 K3S_TOKEN=<mymasternodetoken> sh -```

   ```<mymasternode>``` is k3s-master01 IP Address or hostname

   ```<mymasternodetoken>``` is the token from step 6
   
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

      Access ArgoCD Web UI

      ```
      https://127.0.0.1:8081
   
      ```
   
4. as
