# kubernetes-gitops-lxc
Deploying K3s on lxc containers

 
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
   
8. 
9. as
