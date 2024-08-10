# Encrypt Secrets at rest in Kubernetes

- Create a demo secret
    
    ```bash
    # Create secret with declarative
    kubectl create secret generic app-secret --from-literal=app=hello --dry-run=client -o yaml > app-secret.yaml
    
    kubectl apply -f app-secret.yaml
    ```
    
    ```bash
    # view the secret that is encoded with base64
    kubectl get secrets app-secret -o yaml
    ```
    
- Install “etcd-client” package
    
    ```bash
    sudo apt-get install etcd-client -y
    ```
    
- Retrieve certs and key from etcd pod
    
    ```bash
    kubectl get pods -n <namespace> <etcd_pod> -o jsonpath='{.spec.containers[*].command}' | jq -r
    ```
    
- Get secret data from etcd server (not encrypted)
    
    ```bash
    ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key get /registry/secrets/default/app-secret | hexdump -C
    ```
    
- View the the kubapi server static pod
    
    ```bash
    # list the static pod paht in kubelet service
    systemctl cat kubelet.service
    
    # view the kubelet config file
    cat /var/lib/kubelet/config.yaml
    
    # view the static pod path
    ls -l /etc/kubernetes/manifests
    
    # there will be the static pod of kubeapi server
    cat /etc/kubernetes/manifests/kube-apiserver.yaml
    ```
    
- Create new encryption config file
    
    ```bash
    # Generate random encoded value
    head -c 32 /dev/urandom | base64 
    ```
    
    ```bash
    vim enc.yaml
    ```
    
    ```yaml
    # Create encryption configuration file
    apiVersion: apiserver.config.k8s.io/v1
    kind: EncryptionConfiguration
    resources:
      - resources:
          - secrets
          - configmaps
          - pandas.awesome.bears.example
        providers:
          - aescbc:
              keys:
                - name: key
                  secret: bbaiNeuHhBF2jdE+bP6syVK5bAMSpUF8vscKkyJU97M= # Generated from urandom
          - identity: {}
    ```
    
    ```bash
    # Create directory to store ENC
    mkdir -p /etc/kubernetes/enc
    ```
    
    ```bash
    # Move enc file to the ENC directory
    mv enc.yaml /etc/kubernetes/enc/
    ```
    
- Edit kubeapi server static pod to use ENC
    
    ```bash
    # Edit the kubeapi-server static pod
    vim /etc/kubernetes/manifests/kubeapi-server.yaml
    ```
    
    ```yaml
    # Add volume to the pod
    - name: enc                             
      hostPath:                             
    	  path: /etc/kubernetes/enc          
        type: DirectoryOrCreate
    ```
    
    ```yaml
    # Add volume mount
    - name: enc                           
      mountPath: /etc/kubernetes/enc      
    	  readOnly: true
    ```
    
    ```yaml
    # Add command argument like
    - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
    ```
    
    > Save and exit the static pod file and it’ll reload the file for that static pod.
    > 
    
- Create new secret and verify
    
    ```bash
    # Create secret with declarative
    kubectl create secret generic app-secret2 --from-literal=app=hello --dry-run=client -o yaml > app-secret2.yaml
    
    kubectl apply -f app-secret2.yaml
    ```
    
    ```bash
    # Retrieve value from the newly created secret after ENC applied
    ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key get /registry/secrets/default/app-secret2 | hexdump -C
    ```
    
    ```bash
    # Encrypt all existing secrets with ENC
    kubectl get secrets --all-namespaces -o json | kubectl replace -f -
    ```
