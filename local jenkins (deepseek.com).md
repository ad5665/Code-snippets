# Prompt
```
I need to run jenkins within a local k3d Kubernetes cluster as a pod
When deploying jenkins, i want to provide it my private ssh key so that it can pull from bitbucket
Provide the code to do this
```

To deploy Jenkins in a local `k3d` Kubernetes cluster and provide it with your private SSH key for pulling from Bitbucket, you can follow these steps:

### 1. Create a Kubernetes Secret for the SSH Key
First, you need to create a Kubernetes Secret that contains your private SSH key. This secret will be mounted into the Jenkins pod.

```bash
kubectl create secret generic jenkins-ssh-key \
  --from-file=id_rsa=/path/to/your/private/key \
  --namespace=default
```

Replace `/path/to/your/private/key` with the actual path to your private SSH key.

### 2. Create a Kubernetes ConfigMap for the SSH Config (Optional)
If you need to specify a custom SSH config (e.g., to use a specific SSH key for Bitbucket), you can create a ConfigMap:

```bash
kubectl create configmap jenkins-ssh-config \
  --from-file=config=/path/to/your/ssh/config \
  --namespace=default
```

Replace `/path/to/your/ssh/config` with the actual path to your SSH config file.

### 3. Create a Jenkins Deployment YAML
Create a `jenkins-deployment.yaml` file to define the Jenkins deployment. This file will include the necessary volume mounts for the SSH key and config.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
        - containerPort: 50000
        volumeMounts:
        - name: jenkins-ssh-key
          mountPath: /root/.ssh
          readOnly: true
        - name: jenkins-ssh-config
          mountPath: /root/.ssh/config
          subPath: config
          readOnly: true
      volumes:
      - name: jenkins-ssh-key
        secret:
          secretName: jenkins-ssh-key
          items:
          - key: id_rsa
            path: id_rsa
      - name: jenkins-ssh-config
        configMap:
          name: jenkins-ssh-config
          items:
          - key: config
            path: config
```

### 4. Apply the Jenkins Deployment
Apply the deployment to your `k3d` cluster:

```bash
kubectl apply -f jenkins-deployment.yaml
```

### 5. Expose Jenkins Service (Optional)
If you want to access Jenkins from outside the cluster, you can create a Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30000
  selector:
    app: jenkins
```

Apply the service:

```bash
kubectl apply -f jenkins-service.yaml
```

### 6. Access Jenkins
You can now access Jenkins by navigating to `http://localhost:30000` (or the appropriate IP and port if you're using a different setup).

### 7. Configure Jenkins to Use the SSH Key
Once Jenkins is up and running, you may need to configure it to use the SSH key for pulling from Bitbucket. This can typically be done in the Jenkins UI under the "Credentials" section.

### Summary
- A Kubernetes Secret is created to store your private SSH key.
- A ConfigMap is created to store the SSH config (optional).
- A Jenkins Deployment is created with volume mounts for the SSH key and config.
- Jenkins is deployed and can use the SSH key to pull from Bitbucket.

This setup should allow Jenkins to authenticate with Bitbucket using your private SSH key.
