# Prompt
```
I need to run jenkins within a local k3d Kubernetes cluster as a pod
When deploying jenkins, i want to provide it my private ssh key so that it can pull from bitbucket
Add the git plugin too, this must be install when the pod is starting, by a Jenkins configuration file and with the ssh key, create a jenkins credential for it
Provide the code to do this
```

To achieve this, we need to:
1. Deploy Jenkins in a k3d Kubernetes cluster.
2. Provide Jenkins with your private SSH key for Bitbucket access.
3. Automatically install the Git plugin during Jenkins startup.
4. Automatically create a Jenkins credential for the SSH key using a Jenkins configuration file.

Here’s how you can do it:

---

### Step 1: Prepare the Jenkins Configuration File
We'll use Jenkins Configuration as Code (JCasC) to automate the installation of the Git plugin and the creation of the SSH credential.

Create a `jenkins.yaml` file for JCasC:

```yaml
# jenkins.yaml
jenkins:
  systemMessage: "Jenkins configured automatically by JCasC"
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "admin"
  authorizationStrategy:
    globalMatrix:
      permissions:
        - "Overall/Administer:admin"
  tools:
    git:
      installations:
        - name: "default"
          home: "git"
  credentials:
    system:
      domainCredentials:
        - credentials:
            - basicSSHUserPrivateKey:
                scope: GLOBAL
                id: "bitbucket-ssh-key"
                username: "your-bitbucket-username"
                privateKeySource:
                  directEntry:
                    privateKey: |
                      -----BEGIN RSA PRIVATE KEY-----
                      <your-private-ssh-key-here>
                      -----END RSA PRIVATE KEY-----
unclassified:
  gitPluginConfiguration:
    createAccountBasedOnEmail: false
```

Replace `<your-private-ssh-key-here>` with the contents of your private SSH key (`id_rsa`).

---

### Step 2: Create a Kubernetes ConfigMap for the Jenkins Configuration
Encode the `jenkins.yaml` file in Base64 and create a ConfigMap:

```bash
cat jenkins.yaml | base64 -w 0
```

Create a Kubernetes manifest for the ConfigMap:

```yaml
# jenkins-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jenkins-config
data:
  jenkins.yaml: <base64-encoded-jenkins-yaml>
```

Apply the ConfigMap:

```bash
kubectl apply -f jenkins-config.yaml
```

---

### Step 3: Deploy Jenkins with the Configuration
Create a Kubernetes manifest to deploy Jenkins, mount the ConfigMap, and configure the Git plugin and SSH credential.

```yaml
# jenkins-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
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
        env:
        - name: CASC_JENKINS_CONFIG
          value: /var/jenkins_home/jenkins.yaml
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
        - name: jenkins-config
          mountPath: /var/jenkins_home/jenkins.yaml
          subPath: jenkins.yaml
      volumes:
      - name: jenkins-home
        emptyDir: {}
      - name: jenkins-config
        configMap:
          name: jenkins-config
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30000
  selector:
    app: jenkins
```

Apply the deployment:

```bash
kubectl apply -f jenkins-deployment.yaml
```

---

### Step 4: Access Jenkins
1. Access Jenkins via the NodePort service:
   - Open your browser and go to `http://localhost:30000`.
   - Unlock Jenkins using the initial admin password (retrieve it from the pod logs using `kubectl logs <jenkins-pod-name>`).

2. Verify the Git plugin is installed:
   - Go to **Manage Jenkins > Manage Plugins** and check if the Git plugin is listed.

3. Verify the SSH credential:
   - Go to **Manage Jenkins > Manage Credentials** and check if the SSH credential for Bitbucket is created.

---

### Step 5: Test the Setup
1. Create a new Jenkins job that uses the Git plugin.
2. Configure the job to use the SSH credential for pulling from Bitbucket.
3. Run the job to verify that Jenkins can successfully clone the repository.

---

### Notes
- Replace `your-bitbucket-username` with your actual Bitbucket username.
- If you want persistent storage for Jenkins, replace the `emptyDir` volume with a PersistentVolumeClaim (PVC).
- Ensure your SSH key has the correct permissions (`chmod 600 id_rsa`).

Let me know if you need further assistance!
