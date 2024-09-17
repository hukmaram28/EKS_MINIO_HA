# EKS_MINIO_HA

TODO: Automate infrastructure creation using IaC.

1. **Create EKS cluster using the following command:**

    ```bash
    eksctl create cluster \
      --name minio-cluster \
      --version 1.27 \
      --region us-east-1 \
      --nodegroup-name linux-nodes \
      --node-type t3.medium \
      --nodes 4 \
      --nodes-min 4 \
      --nodes-max 6 \
      --managed
    ```

2. **Verify the cluster creation:**

    ```bash
    kubectl get nodes
    ```

3. **Install Helm locally.**

4. **Create MinIO namespace:**

    ```bash
    kubectl create namespace minio
    ```

5. **Connect to EKS nodes and execute the following commands (necessary to run Longhorn):**

    ```bash
    sudo apt update
    sudo apt install open-iscsi
    ```

6. **Install Longhorn (modify `values.yaml` if required):**

    ```bash
    helm install minio minio/minio \
      --namespace minio \
      --set replicas=4 \
      --set persistence.size=3Gi
    ```

    ![longhorn](./images/longhorn.png)

7. **Deploy MinIO operator:**

    ```bash
    helm repo add minio-operator https://operator.min.io
    helm search repo minio-operator
    helm install \
      --namespace minio-operator \
      --create-namespace \
      operator minio-operator/operator
    ```

8. **Modify Longhorn settings to use Longhorn as the default `storageClass` and set the replica count to 4.**

9. **Install MinIO tenant in HA mode (MinIO in HA needs a minimum of 4 nodes). Modify `values.yaml` and run:**

    ```bash
    helm repo add minio-operator https://operator.min.io
    helm install \
    --namespace minio \
    --create-namespace \
    --values minio-values.yaml \
    minio minio-operator/tenant
    ```

    MinIO dashboard can be accessed at the LoadBalancer endpoint.

    ![minio-ui](./images/minio-ui.png)

10. **Install NGINX Ingress Controller:**

    ```bash
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm install nginx-ingress ingress-nginx/ingress-nginx \
      --namespace ingress-nginx \
      --create-namespace \
      --set controller.service.type=LoadBalancer
    ```

11. **Verify NGINX installation:**

    ```bash
    kubectl get services -o wide -w -n ingress-nginx
    ```

12. **Create three CNAME records for MinIO HL, MinIO console, and Longhorn in the Route 53 hosted zone and point them to the NGINX LoadBalancer.**

    ![dns-entries](./images/dns-entries.png)

13. **Create Kubernetes secrets for SSL certificates. We are using OpenSSL to generate self-signed certificates (We could also create a single certificate for all subdomains):**

    ```bash
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout longhorn-tls.key -out longhorn-tls.crt -subj "/CN=longhorn.boxoffice.guru/O=minio"
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout minio-tls.key -out minio-tls.crt -subj "/CN=minio.boxoffice.guru/O=minio"
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout console.minio-tls.key -out console.minio-tls.crt -subj "/CN=console.minio.boxoffice.guru/O=minio"

    kubectl create secret tls longhorn-tls --cert=longhorn-tls.crt --key=longhorn-tls.key -n longhorn-system
    kubectl create secret tls minio-tls --cert=minio-tls.crt --key=minio-tls.key -n minio
    kubectl create secret tls console-minio-tls --cert=console.minio-tls.crt --key=console.minio-tls.key -n minio
    ```

14. **We can also handle SSL termination at the AWS ELB level instead of NGINX.**

15. **Write ingress resources for Longhorn GUI, MinIO console, and MinIO HL. Apply ingress resources using the following commands:**

    ```bash
    kubectl apply -f ingress-longhorn-ss.yaml
    kubectl apply -f ingress-minio-ss.yaml
    kubectl get ingress -n longhorn-system
    kubectl get ingress -n minio
    ```

16. **When using self-signed certificates, browsers don't trust them and display warnings. Configure a trusted CA (Let's Encrypt):**

    ```bash
    kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.7.1/cert-manager.yaml
    kubectl apply -f cluster-issuer.yaml 
    kubectl apply -f ingress-longhorn-ca.yaml
    kubectl apply -f ingress-minio-ca.yaml 
    ```

    Make sure to delete previously created self-signed secrets.

17. **Reload NGINX Controller:**

    ```bash
    kubectl rollout restart deployment nginx-ingress-ingress-nginx-controller -n ingress-nginx
    ```

    ![minio](./images/minio.png)
    ![longhorn](./images/longhorn-ssl.png)

18. **Install MinIO client locally on Mac:**

    ```bash
    brew install minio/stable/mc
    mc --help
    ```

19. **Create MinIO access key and secret using the console.**

20. **Configure MinIO client:**

    ```bash
    bash +o history
    mc alias set myminio https://minio.boxoffice.guru ACCESS_KEY SECRET_KEY
    bash -o history
    ```

21. **Create a bucket and upload a sample file to it:**

    ```bash
    mc mb myminio/sample-bucket
    touch sample_file.txt
    mc cp sample_file.txt myminio/sample-bucket
    ```

    ![minio-console](./images/minio-console.png)


22. **Cleanup:**

    ```bash
    kubectl delete svc nginx-ingress-ingress-nginx-controller -n ingress-nginx
    eksctl delete cluster --name minio-cluster
    ```