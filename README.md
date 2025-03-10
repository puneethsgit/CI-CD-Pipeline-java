# **CI/CD Pipeline with Jenkins, Docker, and Kubernetes**

## **Overview**
This project demonstrates a **CI/CD pipeline** using **Jenkins, Docker, and Kubernetes** to build, test, and deploy  **Spring Boot application**.

## **Pipeline Workflow**
1. **Checkout Code** from GitHub.
2. **Build & Test** the application using Maven.
3. **Build & Push Docker Image** to Docker Hub.
4. **Update Kubernetes Deployment File** with the new image version.
5. **Deploy the Application** in a Kubernetes Cluster

---

## **Jenkins Pipeline (`Jenkinsfile`)**

```groovy
pipeline {
  agent {
    docker {
      image 'abhishekf5/maven-abhishek-docker-agent:v1'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }
  stages {
    stage('Checkout') {
      steps {
        sh 'echo passed'
        //git branch: 'main', url: 'https://github.com/puneethsgit/Java-Applcation-CI-CD'
      }
    }
    stage('Build and Test') {
      steps {
        sh 'ls -ltr'
        sh 'cd spring-boot-app && mvn clean package'
      }
    }
    stage('Build and Push Docker Image') {
      environment {
        DOCKER_IMAGE = "puneeth11/newultimate-cicd-pipeline:${BUILD_NUMBER}"
        REGISTRY_CREDENTIALS = credentials('docker-cred')
      }
      steps {
        script {
            sh 'cd spring-boot-app && docker build -t ${DOCKER_IMAGE} .'
            def dockerImage = docker.image("${DOCKER_IMAGE}")
            docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
                dockerImage.push()
            }
        }
      }
    }
    stage('Update Deployment File') {
      environment {
        GIT_REPO_NAME = "Java-Applcation-CI-CD"
        GIT_USER_NAME = "puneethsgit"
      }
      steps {
        withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
          sh '''
            git config user.email "princepuneeths814@gmail.com"
            git config user.name "puneethsgit"
            sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" spring-boot-app-manifests/deployment.yml
            git add spring-boot-app-manifests/deployment.yml
            git commit -m "Update deployment image to version ${BUILD_NUMBER}"
            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
          '''
        }
      }
    }
  }
}
```

In your Jenkins pipeline, the following section defines the **agent** that Jenkins will use to execute the pipeline:

```groovy
agent {
    docker {
      image 'abhishekf5/maven-abhishek-docker-agent:v1'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }
```

### **Breaking it Down:**
1. **`agent { docker { ... } }`**  
   - This tells Jenkins to use a **Docker container** as the agent instead of running the pipeline directly on the Jenkins host.

2. **`image 'abhishekf5/maven-abhishek-docker-agent:v1'`**  
   - Specifies that Jenkins should pull and run the Docker image `abhishekf5/maven-abhishek-docker-agent:v1`.
   - This image likely contains all dependencies required for building and testing the application (such as Maven, Java, and Docker CLI).

3. **`args '--user root -v /var/run/docker.sock:/var/run/docker.sock'`**  
   - `--user root`: Runs the container as the **root** user. This is required in some cases when the container needs elevated permissions.
   - `-v /var/run/docker.sock:/var/run/docker.sock`: Mounts the Docker daemon socket from the host into the container.  
     - This allows the Jenkins agent inside the container to communicate with the host’s Docker engine.  
     - This setup enables **Docker-in-Docker (DinD)**, meaning the pipeline can build and push Docker images.

### **Why Is This Used?**
- Running Jenkins agents inside a Docker container ensures a **clean, isolated, and reproducible environment** for builds.
- The mounted Docker socket allows the containerized Jenkins agent to **build and push Docker images** without needing a separate Docker installation inside the container.

### **Why Use Docker as an Agent in Jenkins?**
Instead of running the Jenkins pipeline directly on the host machine, you can use a **Docker container** as an agent. Here’s why this is beneficial:

---

### ** Ensures a Clean, Isolated Build Environment**
- Every pipeline execution runs inside a **fresh container** based on the specified Docker image.
- Avoids dependency conflicts between different builds.
- Prevents build failures caused by host system changes.


### **Why Not Run Directly on the Jenkins Host?**
✅ Running directly on the host **works** but has downsides:
❌ Dependency conflicts between different builds.  
❌ Pollutes the Jenkins host with different tool installations.  
❌ Harder to maintain and upgrade environments.  
❌ Security risks if multiple jobs modify system files.  

---

### **When Should You Use Docker as an Agent?**
✅ When your builds need a **customized runtime** (specific versions of Java, Node.js, Python, etc.).  
✅ When you want to **run builds in isolated environments** to prevent conflicts.  
✅ When your pipeline involves **building and pushing Docker images**.  
✅ When you want **portability** and the same build environment everywhere.  


### **Pipeline Explanation**
| **Stage** | **Purpose** |
|------------|------------|
| Checkout | Fetches the latest code from GitHub. |
| Build & Test | Compiles the Java project using Maven and runs tests. |
| Build & Push Docker Image | Creates a Docker image and pushes it to Docker Hub. |
| Update Deployment File | Updates `deployment.yml` with the new image tag and pushes it to GitHub. |

---

## **Dockerfile (`Dockerfile`)**

```dockerfile
FROM adoptopenjdk/openjdk11:alpine-jre
ARG artifact=target/spring-boot-web.jar
WORKDIR /opt/app
COPY ${artifact} app.jar
ENTRYPOINT ["java","-jar","app.jar"]
```

### **Dockerfile Explanation**
- Uses `openjdk11:alpine-jre` as the base image.
- Copies the built JAR file into the container.
- Runs the JAR file using Java.

---

## **Kubernetes Deployment (`deployment.yml`)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
  labels:
    app: spring-boot-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
    spec:
      containers:
      - name: spring-boot-app
        image: puneeth11/newultimate-cicd-pipeline:2
        ports:
        - containerPort: 8080
```

### **Deployment File Explanation**
- **Creates a deployment with 2 replicas**.
- **Uses the Docker image** (`puneeth11/newultimate-cicd-pipeline:2`).
- **Exposes port 8080** for the application.
- **Automatically updates the image tag** during deployment.

---

## **Kubernetes Service (`service.yml`)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-app-service
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: spring-boot-app
```

### **Service File Explanation**
- Exposes the application using **NodePort**.
- Routes traffic from **port 80 → 8080** inside the container.
- Selects pods labeled `app: spring-boot-app`.

---

## **How Everything Works Together**
1. **Jenkins builds and tests the Spring Boot application**.
2. **Docker builds and pushes the image to Docker Hub**.
3. **Kubernetes pulls the new image and updates the deployment**.
4. **The application is exposed using a Kubernetes Service**.

---

## **Deployment Steps**
### **1. Run Jenkins Pipeline**
- Ensure Jenkins is running and has Docker permissions.
- Configure **GitHub and Docker credentials** in Jenkins.
- Trigger a build to execute the pipeline.

### **2. Apply Kubernetes Files**
```sh
kubectl apply -f deployment.yml
kubectl apply -f service.yml
```
### **3. Access the Application**
```sh
minikube service spring-boot-app-service --url
```
OR
```sh
kubectl get svc
```

---
## **Summary**
| **Component** | **Description** |
|------------|------------|
| Jenkins | Automates CI/CD pipeline execution. |
| Docker | Builds and pushes the application container. |
| Kubernetes | Deploys and manages the application. |
| NodePort Service | Exposes the application externally. |

This ensures a fully automated CI/CD pipeline for **continuous deployment**. 🚀

# SonarQube Setup

## **🔑 Generating and Configuring SonarQube Token**
1. Generate a **SonarQube token** from the SonarQube UI.
2. Add this token as a **SonarQube credential** in Jenkins.
3. Install the **SonarQube Scanner plugin** in Jenkins.

---

## **🔍 Why Is SonarQube Failing in Jenkins?**
Your SonarQube instance is **running and accessible at `http://localhost:9000`**, but Jenkins Cannot reach it.

---

## **✅ Solution: Use the Correct SonarQube URL**
Modify the **SonarQube stage** in your `Jenkinsfile` to use the **host machine's IP (private IP)** instead of `localhost`.

### **🔹 Step 1: Find Your Host Machine's IP**
Run the following command on the host where Jenkins is running:
```bash
ip a | grep inet
```
Example output:
```
inet 192.XXX.X.1X0/24 brd 192.168.1.255 scope global eth0
```
🔹 **Use the extracted IP (e.g., `192.1XX.1.XXX`) in your Jenkins configuration.**

---

### **🔹 Step 2: Update the `Jenkinsfile`**
Modify the SonarQube stage in your `Jenkinsfile`:
```groovy
stage('Static Code Analysis') {
    environment {
        SONAR_URL = "http://192.XXX.1.100:9000" // Use your host machine's IP
    }
    steps {
        withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_AUTH_TOKEN')]) {
            sh 'cd spring-boot-app && mvn sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.host.url=${SONAR_URL}'
        }
    }
}
```
🔹 **Replace `192.168.1.100` with your actual host machine IP.**

---

### **🔹 Step 3: Restart Jenkins and Retry the Pipeline**
Restart Jenkins to apply changes:
```bash
sudo systemctl restart jenkins
```
Then, rerun your pipeline.


🚀 **Your SonarQube setup should now work smoothly with Jenkins!**



# **Explanation of Jenkins Stages: "Build and Push Docker Image" & "Update Deployment File"**  

These two stages handle **containerization, pushing the image to Docker Hub, and updating the Kubernetes deployment file** in GitHub.  

---

## **1️⃣ Stage: "Build and Push Docker Image"**  
This stage is responsible for **building the Docker image** from the JAR file and pushing it to Docker Hub.

### **Environment Variables**  
```groovy
environment {
    DOCKER_IMAGE = "puneeth11/newultimate-cicd-pipeline:${BUILD_NUMBER}"
    REGISTRY_CREDENTIALS = credentials('docker-cred')
}
```
- **`DOCKER_IMAGE`** – Defines the image name and tag (`BUILD_NUMBER` ensures each build gets a unique tag).  
  - Example: `puneeth11/newultimate-cicd-pipeline:10` (if `BUILD_NUMBER = 10`)  
- **`REGISTRY_CREDENTIALS`** – Retrieves Docker Hub credentials from Jenkins using `credentials('docker-cred')`.  

### **Steps (Building & Pushing the Image)**  
```groovy
script {
    sh 'cd spring-boot-app && docker build -t ${DOCKER_IMAGE} .'
    def dockerImage = docker.image("${DOCKER_IMAGE}")
    docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
        dockerImage.push()
    }
}
```

#### **Step 1: Build Docker Image**
```sh
cd spring-boot-app && docker build -t ${DOCKER_IMAGE} .
```
- Moves into the `spring-boot-app` directory.  
- Builds a Docker image using the `Dockerfile`.  
- The image is tagged as `"puneeth11/newultimate-cicd-pipeline:${BUILD_NUMBER}"`.  

#### **Step 2: Push Docker Image to Docker Hub**
```groovy
def dockerImage = docker.image("${DOCKER_IMAGE}")
docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
    dockerImage.push()
}
```
- Uses Jenkins' `docker.image()` to reference the newly built image.  
- `docker.withRegistry()` authenticates with Docker Hub using `docker-cred`.  
- Pushes the image to the Docker Hub repository **"puneeth11/newultimate-cicd-pipeline"** with the **`${BUILD_NUMBER}`** tag.  

**Result:**  
The image is now stored in Docker Hub and can be pulled by Kubernetes.

---

## **2️⃣ Stage: "Update Deployment File"**  
This stage **updates the Kubernetes deployment file (`deployment.yml`)** to use the newly built Docker image.  

### **Environment Variables**  
```groovy
environment {
    GIT_REPO_NAME = "Java-Applcation-CI-CD"
    GIT_USER_NAME = "puneethsgit"
}
```
- **`GIT_REPO_NAME`** – Name of the GitHub repository.  
- **`GIT_USER_NAME`** – GitHub username.  

### **Steps (Updating Kubernetes YAML File in GitHub)**  
```groovy
withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
    sh '''
        git config user.email "princepuneeths814@gmail.com"
        git config user.name "puneethsgit"
        BUILD_NUMBER=${BUILD_NUMBER}
        sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" spring-boot-app-manifests/deployment.yml
        git add spring-boot-app-manifests/deployment.yml
        git commit -m "Update deployment image to version ${BUILD_NUMBER}"
        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
    '''
}
```

#### **Step 1: Authenticate with GitHub**  
```groovy
withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')])
```
- Fetches **GitHub credentials** stored in Jenkins (`github` ID).  
- Stores the GitHub token in the `GITHUB_TOKEN` variable.  

#### **Step 2: Configure Git User Details**  
```sh
git config user.email "princepuneeths814@gmail.com"
git config user.name "puneethsgit"
```
- Configures Git **user details** (used for committing changes).  

#### **Step 3: Update Deployment File with New Image**  
```sh
sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" spring-boot-app-manifests/deployment.yml
```
- **Finds and replaces** `replaceImageTag` in `deployment.yml` with `${BUILD_NUMBER}`.  
- Example: If `BUILD_NUMBER=10`, the YAML file updates to:  
  ```yaml
  image: puneeth11/newultimate-cicd-pipeline:10
  ```
  This ensures Kubernetes pulls the latest image.

#### **Step 4: Commit & Push Changes to GitHub**
```sh
git add spring-boot-app-manifests/deployment.yml
git commit -m "Update deployment image to version ${BUILD_NUMBER}"
git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
```
- **Adds the updated file** to Git.  
- **Commits the change** with a message: `"Update deployment image to version ${BUILD_NUMBER}"`.  
- **Pushes the changes** to the GitHub repository.  

---

### **Final Result**
✔ Docker image is built & pushed to Docker Hub.  
✔ `deployment.yml` is updated with the latest image tag.  
✔ GitHub repository is updated.  
✔ Kubernetes will now use the latest image when redeploying the application. 

# **Explanation of the `sed` Command:**
```sh
sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" spring-boot-app-manifests/deployment.yml
```
This command **modifies the `deployment.yml` file** by replacing `replaceImageTag` with the actual **`${BUILD_NUMBER}`** value.

---

### **Breaking It Down:**
1. **`sed`** → Stream Editor, used for text manipulation.  
2. **`-i`** → Edits the file **in place** (modifies the file directly).  
3. **`s/replaceImageTag/${BUILD_NUMBER}/g`**  
   - **`s/`** → Substitutes text.  
   - **`replaceImageTag`** → The placeholder text to be replaced in the `deployment.yml` file.  
   - **`${BUILD_NUMBER}`** → The Jenkins build number (e.g., `10`, `11`, `12` …).  
   - **`/g`** → **Global replacement**, ensuring all occurrences of `replaceImageTag` are replaced.  
4. **`spring-boot-app-manifests/deployment.yml`** → The file being modified.

---

### **Example:**
#### **Before Modification (`deployment.yml`):**
```yaml
containers:
  - name: spring-boot-app
    image: puneeth11/newultimate-cicd-pipeline:replaceImageTag
    ports:
      - containerPort: 8080
```

#### **Jenkins Runs: (`BUILD_NUMBER=10`)**
```sh
sed -i "s/replaceImageTag/10/g" spring-boot-app-manifests/deployment.yml
```

#### **After Modification (`deployment.yml`):**
```yaml
containers:
  - name: spring-boot-app
    image: puneeth11/newultimate-cicd-pipeline:10
    ports:
      - containerPort: 8080
```

---

### **Why is this Important?**
✔ **Ensures Kubernetes always pulls the latest image** when a new deployment happens.  
✔ **Automates the process** of updating the deployment without manual edits.  
✔ Works **dynamically with Jenkins**, as each build gets a unique number (`BUILD_NUMBER`).  

# ArgoCD Setup for Continuous Delivery in Jenkins Pipeline

## Prerequisites
- A running **Minikube** cluster
- `kubectl` installed and configured
- `helm` installed
- `minikube status` should show that the cluster is running

---

## 1. Install ArgoCD Using OperatorHub.io Documentation
ArgoCD can be installed using the OperatorHub.io method or by applying a YAML file.

### Option 1: Install via OperatorHub.io
Follow the instructions from [OperatorHub.io](https://operatorhub.io/operator/argocd).

### Option 2: Install via YAML Manifest
Create a YAML file (`argocd-basic.yml`) and copy the following contents:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: example-argocd
  labels:
    example: basic
spec:
  server:
    service:
      type: NodePort
```

Apply the YAML file:
```sh
kubectl apply -f argocd-basic.yml
```

Verify ArgoCD installation:
```sh
kubectl get pods -n argocd
```

---

## 2. Change ArgoCD Service Type
By default, ArgoCD is set to `ClusterIP`. To access it externally, change it to `NodePort`.

Edit the service:
```sh
kubectl edit svc example-argocd-server -n argocd
```
Change:
```yaml
spec:
  type: NodePort  # Change from ClusterIP to NodePort
```
Save and exit.

Expose ArgoCD:
```sh
minikube service example-argocd-server -n argocd
```
This will return a URL. Open it in your browser to access ArgoCD.

---

## 3. Login to ArgoCD
Retrieve the admin password:
```sh
kubectl get secret example-argocd-cluster -n argocd -o jsonpath="{.data.admin\.password}" | base64 -d
```

- **Username:** `admin`
- **Password:** `<decoded password>`

Login to ArgoCD:
```sh
argocd login <ARGOCD_URL>:<NODEPORT> --username admin --password <decoded password>
```

---

## 4. Create a New Project in ArgoCD
1. Open the ArgoCD UI in a browser using the provided Minikube URL.
2. Click **New Project**.
3. Give it a name and assign a namespace (`default` if unsure).
4. Select the **source code repository** (GitHub repository containing Kubernetes manifests).
5. Select the **destination cluster** (Minikube cluster suggestion).
6. Click **Create**.

Verify the project:
```sh
kubectl get deploy -n argocd
kubectl get pods -n argocd
```

---

## 5. Deploy and Access the Application
ArgoCD will now automatically sync and deploy applications.

To check deployment:
```sh
kubectl get deploy -n default
kubectl get pods -n default
```

To access the Spring Boot application:
```sh
minikube service spring-boot-app-server-service -n default
```
This will return a URL with a **NodePort**, which you can open in your browser to access the deployed application.

---

## 🎉 Done!
You have successfully set up **ArgoCD for Continuous Delivery** in your Minikube environment.. 🚀


