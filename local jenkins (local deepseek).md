# Prompt 
```
I need to run jenkins within a local k3d Kubernetes cluster as a pod
When deploying jenkins, i want to provide it my private ssh key so that it can pull from bitbucket
Add the git plugin too and with the ssh key, create a jenkins credential for it
Provide the code to do this
```

To run Jenkins within a local k3d Kubernetes cluster as a pod and configure it to use your private SSH key for pulling from Bitbucket, you'll need to follow these steps:

1. **Create a Local Kubernetes Cluster with k3d**: If you haven't already set up a local Kubernetes cluster using k3d, you can do so by running the following command:
    ```sh
    k3d cluster create my-cluster
    ```

2. **Deploy Jenkins as a Pod**: You need to create a Kubernetes deployment and service for Jenkins. Here's an example of a `deployment.yaml` file that includes the necessary configurations:

    ```yaml
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
              name: http
            - containerPort: 50000
              name: jnlp
            volumeMounts:
            - name: ssh-key
              mountPath: /var/jenkins_home/.ssh
          volumes:
          - name: ssh-key
            secret:
              secretName: jenkins-ssh-key
    ```

3. **Create a Service for Jenkins**: Additionally, you need to create a service to expose the Jenkins pod externally:

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: jenkins
    spec:
      selector:
        app: jenkins
      ports:
        - protocol: TCP
          port: 80
          targetPort: 8080
    ```

4. **Apply the Configurations**: Deploy the Jenkins pod and service using `kubectl`:

    ```sh
    kubectl apply -f deployment.yaml
    kubectl apply -f service.yaml
    ```

5. **Expose the Jenkins UI**: To access the Jenkins UI, you can forward a local port to the Jenkins service:

    ```sh
    kubectl port-forward svc/jenkins 8080:80
    ```

6. **Create an SSH Key Secret**: You need to create a Kubernetes secret that contains your private SSH key. This can be done using `kubectl create secret generic`:

    ```sh
    kubectl create secret generic jenkins-ssh-key --from-file=id_rsa=/path/to/your/private/key
    ```

7. **Configure Jenkins to Use the SSH Key**: After accessing the Jenkins UI, you need to configure it to use the SSH key for pulling from Bitbucket:

    - Go to `Manage Jenkins` > `System Log In`.
    - Click on `Add credentials` under the `Credentials` section and select `SSH Username with private key`.
    - Enter your SSH key details.
    - Save the credential.

8. **Install the Git Plugin**: In the Jenkins UI, go to `Manage Jenkins` > `Manage Plugins` and search for `Git plugin`. Install it and restart Jenkins if required.

9. **Configure a Job to Use the SSH Key**: You can now create a new job in Jenkins that uses the SSH key for pulling from Bitbucket:

    - Go to `New Item` and enter your project name.
    - Select `Freestyle Project`.
    - In the `Source Code Management` section, select `Git`.
    - Enter your repository URL and credentials ID (which should be the same SSH key you created).
    - Save the job configuration.

By following these steps, you will have Jenkins running in a Kubernetes pod on your local k3d cluster, configured to use your private SSH key for pulling from Bitbucket, and installed the Git plugin with the necessary credentials.
