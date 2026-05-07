# Jenkins + SonarQube Setup and Pipeline


## 1. Create EC2 Instances
- **Instance 1:** Jenkins (instance type : c7i-flex.large or more) 
- **Instance 2:** SonarQube  (instance type : c7i-flex.large or more)

## 2. Setup Jenkins EC2 Server

### Go To Root 
```sh
sudo -i
```

### Update Instance
```sh
apt update
```
### Install Java 17
```bash
apt install openjdk-17-jdk -y
```

### Install Jenkins

Follow official Jenkins documentation
👉 [Jenkins Installation](https://www.jenkins.io/doc/book/installing/linux/)

### Install Maven
```bash
apt install maven -y
```

## 3. Setup SonarQube EC2 Server

### Go To Root 
```sh
sudo -i
```

### Update Instance
```sh
apt update
```
### Install Docker
```bash
apt install docker.io -y
```
### Run SonarQube
```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
### Git Clone
```sh
git clone https://github.com/Rohit-1920/EasyCRUD.git
```
### Update the pom.xml with sonarqube dependancy
```sh
cd EasyCRUD/backend/
```
```sh
ls
```

### pom.xml with Sonar Maven Plugin
👉 [maven sonar plugin](https://mvnrepository.com/artifact/org.sonarsource.scanner.maven/sonar-maven-plugin)

```sh
rm pom.xml
```
```sh
ls
```
```sh
nano pom.xml
```

## 4. Access Jenkins & SonarQube
- Jenkins: `http://<jenkins-public-ip>:8080`
- SonarQube: `http://<sonarqube-public-ip>:9000`
    - Login SonarQube with:
        - Username: `admin`
        - Password: `admin`
        - Change the default password after first login.


## 5. Configure SonarQube

1. Create Webhook
    - Go to: **Administration** → **Configuration** → **Webhooks** → **Create**
    - Name: `Sonar-webhook`
    - URL: `http://<jenkins-public-ip>:8080/sonarqube-webhook/`
2. Create a Project
    - Go to **Projects** → **Create Project** → **Local Project*
    - Project display Name: `studentapp`
    - Project key: `studentapp`
    - Main branch name: `main` then click **Next**
    - Select **Use global settings**
    - Generate **token** → **Copy** & save it
    - Select **Maven** as build tool → Copy the given command

## 6. Configure Jenkins
### 1. Install Plugin
  - Dashboard → Manage Jenkins → Plugins → Available Plugins
  - Install: **SonarQube Scanner for Jenkins**
### 2. Add Credentials
  - Dashboard → Manage Jenkins → Credentials → Global credentials (unrestricted)
  - Add new credential:
      - Kind: `Secret Text`
      - Secret: `<SonarQube Token>`
      - ID: `sonar-token`
### 3. Configure SonarQube Server
  - Dashboard → Manage Jenkins → System
  - Find SonarQube Server section
  - Enable environment variable
  - Add new SonarQube:
      - Name: `Sonar-env`
      - Server URL: `http://<sonarqube-public-ip>:9000`
      - Authentication Token: `sonar-token`
  - Save changes
**Note (optional)**: After configuring, restart the Jenkins server to ensure it operates smoothly. (http://<jenkins-public-ip>:8080/restart)

## 7. Create Jenkins Pipeline
1. Go to Dashboard → New Item → Pipeline
2. Paste the following pipeline code:
```jdp
pipeline {
    agent any

    stages {
        stage('clone repository') {
            steps {
                git branch: 'main', url: 'https://github.com/Rohit-1920/EasyCRUD.git'
            }
        }

        stage('build') {
            steps {
                sh '''
                cd backend
                mvn clean package -DskipTests
                '''
            }
        }

        stage('sonar analysis') {
            steps {
                withSonarQubeEnv(credentialsId: 'sonar-token', installationName: 'Sonar-env') {
                    sh '''
                    cd backend
                    mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                      -Dsonar.projectKey=studentapp \
                      -Dsonar.projectName=studentapp
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('deploy') {
            steps {
                echo 'deployment stage'
            }
        }
    }
}

```

## 8. Run the Pipeline
- Build the job in Jenkins.
- Check results in SonarQube Dashboard.
