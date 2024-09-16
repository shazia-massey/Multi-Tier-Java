pipeline {
    agent any
    
    tools {
        maven  'maven3'
    }
    
    environment{
        SCANNER_HOME= tool 'sonar-scanner'
        IMAGE_NAME = "masseys/bankapp"
        TAG = "${env.BUILD_NUMBER}"
    }
    
    

    stages {
        stage('Git Checkout') {
            steps {
              git credentialsId: 'git-cred', url: 'https://github.com/shazia-massey/Multi-Tier-Java.git'
            }
        }
        
        stage('Compile') {
            steps {
              sh "mvn compile"
            }
        }
        
        stage('Unit tests') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }
        
        stage('Trivy FS Scan ') {
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
               withSonarQubeEnv('sonar') {
                   sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=multitier -Dsonar.projectName=multitier -Dsonar.java.binaries=target"
               } 
            }
        }
        
         stage('Quality GateCheck') {
            steps {
              timeout(time: 1, unit: 'HOURS') {
                     waitForQualityGate abortPipeline: false
                 }
            }
        }
        
         stage('Build') {
            steps {
              sh "mvn package -DskipTests=true"
            }
        }
        
         stage('publish To Nexus') {
            steps {
              withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                  sh "mvn deploy -DskipTests=true"
                }
            }
        }
        
         stage('Build  Docker Image') {
            steps {
                script {
               withDockerRegistry(credentialsId: 'docker-cred') {
                   sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                     
                     }
                 }
            }
        }
        
         
        stage('Trivy Image Scan ') {
            steps {
                sh "trivy image --format table -o image.html ${IMAGE_NAME}:${TAG}"
            }
        }
        
        
         stage('Push Docker Image') {
            steps {
                script {
               withDockerRegistry(credentialsId: 'docker-cred') {
                   sh "docker push  ${IMAGE_NAME}:${TAG}"
                     
                     }
                 }
            }
        }
        
         stage('Update Kubernetes Manifest') {
            steps {
                script {
                    // Replace the Docker image tag in line 58 of the ds.yml file
                    sh """
                    sed -i '58s|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${TAG}|' ds.yml
                    """
                }
            }
        }
        
        
              
           stage('Commit and Push Changes') {
    steps {
        script {
            // Use GitHub credentials from Jenkins
            withCredentials([usernamePassword(credentialsId: 'git-cred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                sh """
                git config --global user.email "masseys.s123@gmail.com"
                git config --global user.name "shazia-massey"
                git remote set-url origin https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/shazia-massey/Multi-Tier-Java.git
                git pull origin master || git pull origin main
                git add ds.yml
                git commit -m "Update image to masseys/bankapp:6"
                git push origin HEAD:main || git push origin HEAD:master
                """
            }
        }
    }
}

        
         stage('Deploy To K8') {
            steps {
              withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://3F5E779C7DF69C01E718C3A9E1325FF8.gr7.us-east-1.eks.amazonaws.com') {
                     sh " kubectl apply -f ds.yml"
                     sleep 30
                }
            }
        }
        
         stage('Verify k8 Deployment') {
            steps {
              withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://3F5E779C7DF69C01E718C3A9E1325FF8.gr7.us-east-1.eks.amazonaws.com') {
                     sh "kubectl get pods -n webapps"
                     sh "kubectl get svc -n webapps"
                }
            }
        }
    }
}
