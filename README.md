# MicroK8s Setup Guide

This guide provides step-by-step instructions to set up MicroK8s, deploy MySQL and phpMyAdmin, and configure port forwarding using MacOS.

## Prerequisites
Ensure the following are installed on your system before proceeding:

### 1. Install `kubectl` (Kubernetes CLI)
`kubectl` is used to manage Kubernetes clusters.

- **Optional:** MicroK8s includes `kubectl` by default.
- If `kubectl` is already installed separately, assign MicroK8s to use the installed version. Follow the steps after installed MicroK8s.

### 2. Install Multipass
MicroK8s runs inside a Multipass VM, which must be installed.

Install Multipass:
  ```sh
  brew install --cask multipass
  ```
Check if it's running properly::
  ```sh
  multipass info
  ```

### 3. Install MicroK8s
Install MicroK8s:

```sh
brew install ubuntu/microk8s/microk8s
```

Once installed, run:

```sh
microk8s install
```

Check if MicroK8s is Running
```sh
microk8s status --wait-ready
```

If it's not running, start it:
```sh
microk8s start
```

If you already have kubectl, you can set it to use MicroK8s by running
```sh
microk8s kubectl config view --raw > ~/.kube/config
```

Now, check if MicroK8s is working:
```sh
kubectl get nodes
```

If you prefer to use MicroK8s' built-in kubectl, you can run:
```sh
microk8s kubectl get nodes
```

### 4. Enable Essential Add-ons
Enable necessary add-ons like the Kubernetes Dashboard, DNS, and storage:

```sh
microk8s enable dashboard dns storage
```

### 5. Port Forward Dashboard and Create Token to Login
To access the Kubernetes Dashboard, port-forward it and generate a login token:

```sh
microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443
```

View config:

```sh
microk8s kubectl config view --raw
```

Get secret generated to verify:

```sh
microk8s kubectl get secrets -n kube-system
```

Generate the token from secret with related namespace:
```sh
microk8s kubectl get secret microk8s-dashboard-token -n kube-system -o jsonpath="{.data.token}" | base64 --decode
```

Access the Dashboard at:
```
https://127.0.0.1:10443
```

### 6. Prepare MySQL YAML File
Create a `mysql-deployment.yaml` file for MySQL deployment:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  my.cnf: |
    [mysqld]
    bind-address = 0.0.0.0

---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  root-password: c3VwZXJzZWNyZXQ= # Base64 encoded 'supersecret'
  mysql-user: bXl1c2Vy # Base64 encoded 'myuser'
  mysql-password: bXlwYXNzd29yZA== # Base64 encoded 'mypassword'

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: mysql-user
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: mysql-password
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
            - name: mysql-config
              mountPath: /etc/mysql/my.cnf
              subPath: my.cnf
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pvc
        - name: mysql-config
          configMap:
            name: mysql-config

---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  type: NodePort  # Expose MySQL outside the cluster
  ports:
    - protocol: TCP
      port: 3306        # MySQL internal port
      targetPort: 3306  # MySQL container port
      nodePort: 30009   # Host accessible port (must be 30000-32767)

```

### 7. Prepare phpMyAdmin YAML File
Create a `phpmyadmin.yaml` file:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpmyadmin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: phpmyadmin
  template:
    metadata:
      labels:
        app: phpmyadmin
    spec:
      containers:
        - name: phpmyadmin
          image: phpmyadmin
          env:
            - name: PMA_HOST
              value: "mysql"  # Connects to MySQL inside Kubernetes
            - name: PMA_PORT
              value: "3306"
          ports:
            - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: phpmyadmin
spec:
  selector:
    app: phpmyadmin
  type: NodePort  # Exposes phpMyAdmin on a high port
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30008  # Change this if needed (must be between 30000-32767)
```

### 8. Apply MySQL and phpMyAdmin Deployments
Run the following commands to apply the configurations:
```sh
microk8s kubectl apply -f mysql-deployment.yaml
microk8s kubectl apply -f phpmyadmin.yaml
```

### 9. Port Forward MySQL and phpMyAdmin
Expose MySQL on port `3309` and phpMyAdmin on `3310`:
```sh
microk8s kubectl port-forward svc/mysql 3309:3306 &
microk8s kubectl port-forward svc/phpmyadmin 3310:80 &
```

### 10. Verify Running Services
Check if the services are running correctly:
```sh
microk8s kubectl get pods
microk8s kubectl get svc -A
```

### 11. Access phpMyAdmin
Open your browser and go to:
```
http://127.0.0.1:3310
```
Login using:
- **Username:** root
- **Password:** supersecret

## Conclusion
You have successfully set up MicroK8s with MySQL and phpMyAdmin, configured port-forwarding, and accessed the services locally. 🚀

