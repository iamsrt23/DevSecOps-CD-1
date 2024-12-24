# CD

**PRODUCTION GRADE DEVSECOPS CICD Pipeline:**

**Traditional Deployment:**

Build & Unit Test. —> Build Image —> kubectl(Yaml)

- Pipeline has access to k8s cluster
- Pipeline cannot redeploy
- Pipeline needs to deploy across QA/Stage/Prod environment

git —> Desired State —> argocd —> actual state —> k8s cluster

1. Create 1 EC2 Server -t2.medium—> minikube server(8Gb is fine)

**Install Minikube:**

```
$ Install Docker
$ sudo apt update && sudo apt -y install docker.io

 Install kubectl
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.23.7/bin/linux/amd64/kubectl && chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/kubectl

 Install Minikube
$ curl -Lo minikube https://storage.googleapis.com/minikube/releases/v1.23.2/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/

 Start Minikube
$  sudo apt install conntrack
$  minikube start --vm-driver=none

```

```
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get pods -n argocd
kubectl get svc -n argocd

$ kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

kubectl get svc -n argocd --> To check which port it's running
kubectl get pods -n argocd --> To check pods are running

minikubeserverpublicdns:NodePort --> argo cd login page

to get the password run below command

For version 1.9 or later:
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

```

**Create kubernetes Folder:**

create deployment.yaml

kubectl create deployment democicd --image=iamsrt23/democicd --port=8080 --replicas=2 --dry-run=client -o yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: democicd
  name: democicd
spec:
  replicas: 2
  selector:
    matchLabels:
      app: democicd
  template:
    metadata:
      labels:
        app: democicd
    spec:
      containers:
        - image: iamsrt23/democicd
          name: democicd
          ports:
            - containerPort: 8080
            
            
```

service.yaml:

kubectl expose deployment democicd --port=8080 --target-port=8080 --type=NodePort --dry-run=client -o yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: democicd
  name: democicd
spec:
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: democicd
  type: NodePort

```

**We Are in Argo WebApp:**

settings —>  Repositories —> + Connect Repo —> 

Choose your connection Method: VIA HTTPS,

Connect REPO USING HTTPS

Type: git,

Project: default

Repository URL: [https://github.com/iamsrt23/DevSecOps-CD-1.git](https://github.com/iamsrt23/DevSecOps-CD-1.git)

username: iamsrt23

password: github-token

**CONNECT**

We need to tell the location so

Goto APPLICATIONS —> NEW APP—>

Application Name: demo

Project: default

SYNC POLICY: Manual

**SOURCE**

Repository URL: [https://github.com/iamsrt23/DevSecOps-CD-1.git](https://github.com/iamsrt23/DevSecOps-CD-1.git)

Revision: Head

path: kubernetes

Destination:

Cluster URL:  https://kubernetes.default.svc

Namespace: default

**CREATE**

SyncStatus —> SYNC

Chcek the PODS

**Production Grade Deployment Pipeline:**

Jenkins Job call it is as a CD pipeline

**Goto Jenkins Create a CD pipeline:**

```groovy
pipeline{
    agent {label 'build'}
    parameters{
        password(name: 'PASSWD', defaultValue: '', description: 'Please Enter Your GitHub Password')
        string(name: 'IMAGETAG', defaultValue: '1',description: 'Please Enter Image Tag to Deploy' )
    }
    stages{
        stage('Deploy'){
            steps{
                git branch: 'main', credentialsId: 'githubtoken', url:'https://github.com/iamsrt23/DevSecOps-CD-1.git'
                dir("./kubernetes"){
                    sh "sed -i 's/image: iamsrt23.*/image: iamsrt23\\/democicd:$IMAGETAG/g' deployment.yaml"

                }
                sh 'git commit -a -m "New Deployment for Build $IMAGETAG'
                sh "git push https://iamsrt23:$PASSWD@gihub.com/iamsrt23/DevSecOps-CD-1.git"

            }
        }
    }
}
```

Build Now :

Give Parameters

PASSWD:

IMAGETAG:

**Till Now CD Part is Completed**

Build Pipeline Triggered —> Deployment Pipeline i.e means (downstream job) one job call another job  

The Pipeline Looks 

```groovy
pipeline{
    agent {label 'build'}
    environment {
        registry = "iamsrt23/democicd"
        registryCredential = "dockerhub"
    }
    parameters {
        password(name: 'PASSWD', defaultValue: '', description: 'Please Enter you GitHub Password')
    }
    stages{
        stage('Checkout'){
            steps{

            }
        }
        stage('Stage1: Build'){
            steps{
                echo "Building Jar Component ..."
                sh "export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64; mvn clean package"
                
            }
        }
        stage('Stage2: Code Coverage'){
            steps{
                echo "Running Code Coverage ...."
                sh  "export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64; mvn jacoco:report "
                
            }
        }
        stage('Stage3 : SCA'){
            steps{
                echo "Running Software Composition Analysis"
                sh  "export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64; mvn org.owasp:dependency-check-maven:check"
                
            }
        }
        stage('Stage4: SAST'){
            steps{
                echo "Running Static Applicatio security testing using SonarQube Scanner...."
                withSonarQubeEnv('mysonarqube'){
                    sh "mvn sonar:sonar -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml -Dsonar.dependencyCheck.jsonReportPath=target/dependency-check-report.json -Dsonar.depencencyCheck.htmlReportPath=target/dependency-check-report.html -Dsonar.projectName='cicd-demo'"
                }
                

            }
        }
        stage('Stage 5: QualityGates'){
            steps{
                echo "Running Quality Gates to verify the code quality"
                script{
                    timeout(time: 1, unit: 'MINUTES'){
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK'){

                            error 'Pipeline aborted due to quality gate failure: ${qg.status}'

                        }
                    }
                }
                
            }
        }
        stage('Stage6: BuildImage'){
            steps{
                echo "Build Docker Image"

                script{
                    docker.withRegistry('',registryCredential){
                        myImage = docker.build registry + " :$BUILD_NUMBER"
                        myImage.push()
                    }
                }

                
            }
        }
        stage('Stage 7: Scan Image'){
            steps{
                echo "Scanning Image for Vulnerabilities"
                sh "trivy image --scanners vuln --offline-scan iamsrt23/democicd:$BUILD_NUMBER > trivyresults.txt"

                
            }
        }
        stage('Stage 8: SmokeTest'){
            steps{
                echo "Smoke test the Image"
                sh "docker run -d --name smokerun -p 8000:8000 iamsrt23/democicd:$BUILD_NUMBER"
                sh "sleep 90; ./check.sh"
                sh "docker rm --force smokerun"
            }
        stage('Stage 9:Trigger Deployment'){
            steps{
                script{
                    TAG = "$BUILD_NUMBER"
                    echo "Trigger CD Pipeline"
                    build wait: false, job: 'demo-cd', parameters: [password(name: 'PASSWD', description: 'Please Enter you GitHub Password', value: params.PASSWD), string(name: 'IMAGETAG', value: TAG)]
                }
            }
        }
        }
    }
}
```