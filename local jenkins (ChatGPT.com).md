# Promt 
```
I need to run jenkins within a local k3d Kubernetes cluster as a pod
When deploying jenkins, i want to provide it my private ssh key so that it can pull from bitbucket
Add the git plugin too, this must be install when the pod is starting, by a Jenkins configuration file and with the ssh key, create a jenkins credential for it
Provide the code to do this
```

To achieve this, you'll need to:  

1. **Set up a `k3d` cluster** if you haven’t already.  
2. **Deploy Jenkins as a pod** in your local Kubernetes cluster.  
3. **Mount your private SSH key** into the Jenkins pod securely.  
4. **Install the Git plugin** when Jenkins starts.  
5. **Use a Jenkins configuration-as-code (`jcasc`) file** to create a credential for the SSH key automatically.  

---

Here’s the full setup:  

1. **Create a `k3d` cluster:**  
   ```sh
   k3d cluster create jenkins-cluster
   ```

2. **Prepare a Kubernetes secret for the SSH key:**  
   ```sh
   kubectl create secret generic jenkins-ssh-key --from-file=id_rsa=$HOME/.ssh/id_rsa
   ```

3. **Create the Jenkins deployment with the Git plugin and SSH key**  
   Below is the complete YAML to deploy Jenkins, configure the Git plugin, and set up SSH credentials in Jenkins.  

---

### **Jenkins Deployment and Configurations**
I'll provide the YAML files required.

#### **1. `jenkins-deployment.yaml`**
Creates a Jenkins pod with a volume for the SSH key and installs the Git plugin.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  labels:
    app: jenkins
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
            - name: jenkins-home
              mountPath: /var/jenkins_home
            - name: ssh-key
              mountPath: /var/jenkins_home/.ssh
              subPath: id_rsa
              readOnly: true
          env:
            - name: JAVA_OPTS
              value: "-Djenkins.install.runSetupWizard=false"
          lifecycle:
            postStart:
              exec:
                command:
                  - "/bin/sh"
                  - "-c"
                  - |
                    echo "Installing Git plugin..."
                    curl -L --silent --output /usr/share/jenkins/ref/plugins/git.hpi \
                    https://updates.jenkins.io/latest/git.hpi
      volumes:
        - name: jenkins-home
          emptyDir: {}
        - name: ssh-key
          secret:
            secretName: jenkins-ssh-key
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: jenkins
  type: NodePort
```

---

#### **2. `jenkins-casc-config.yaml`**
This file configures Jenkins on startup, installing the Git plugin and setting up SSH credentials.

```yaml
jenkins:
  systemMessage: "Jenkins configured automatically by Configuration as Code plugin"
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "admin"
  authorizationStrategy: "loggedInUsersCanDoAnything"
  credentials:
    system:
      domainCredentials:
        - credentials:
            - basicSSHUserPrivateKey:
                id: "bitbucket-ssh"
                scope: GLOBAL
                description: "SSH Key for Bitbucket"
                username: "git"
                privateKeySource:
                  directEntry:
                    privateKey: ${SSH_PRIVATE_KEY}
```

---

#### **3. `jenkins-configmap.yaml`**
This makes the `jenkins-casc-config.yaml` available inside the Jenkins pod.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jenkins-config
data:
  jenkins-casc-config.yaml: |
    jenkins:
      systemMessage: "Jenkins configured automatically by Configuration as Code plugin"
      securityRealm:
        local:
          allowsSignup: false
          users:
            - id: "admin"
              password: "admin"
      authorizationStrategy: "loggedInUsersCanDoAnything"
      credentials:
        system:
          domainCredentials:
            - credentials:
                - basicSSHUserPrivateKey:
                    id: "bitbucket-ssh"
                    scope: GLOBAL
                    description: "SSH Key for Bitbucket"
                    username: "git"
                    privateKeySource:
                      directEntry:
                        privateKey: ${SSH_PRIVATE_KEY}
```

---

#### **4. `jenkins-secret.yaml`**
To provide the private key to Jenkins.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-ssh-key
type: Opaque
data:
  id_rsa: | 
    <base64-encoded-private-key>
```
Encode your private SSH key using:
```sh
cat ~/.ssh/id_rsa | base64
```
Then replace `<base64-encoded-private-key>` with the output.

---

### **Deploy everything**
Apply the configurations in this order:

```sh
kubectl apply -f jenkins-secret.yaml
kubectl apply -f jenkins-configmap.yaml
kubectl apply -f jenkins-deployment.yaml
```

---

### **Access Jenkins**
To get the Jenkins admin password:
```sh
kubectl exec -it $(kubectl get pod -l app=jenkins -o jsonpath="{.items[0].metadata.name}") -- cat /var/jenkins_home/secrets/initialAdminPassword
```
Then, access Jenkins via:
```sh
kubectl port-forward svc/jenkins-service 8080:8080
```
Open [http://localhost:8080](http://localhost:8080) in your browser.

---

### **Verify Git Plugin & SSH Key**
- Go to **Manage Jenkins → Plugins** to check if the Git plugin is installed.
- Go to **Manage Jenkins → Credentials** to confirm the SSH key exists.

This setup ensures Jenkins has the Git plugin installed, uses your SSH key for Bitbucket, and stores it as a credential. Let me know if you need modifications! 🚀
