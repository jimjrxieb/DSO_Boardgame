pipeline {
    agent any
    tools {
        jdk "jdk17"
        maven "maven3"
    }
    
    environment {
        SCANNER_HOME= tool 'sonar-scanner'
	GG_API_KEY = credentials('gitguard_cred')
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/jimjrxieb/DSO_Boardgame.git'
            }
        }
        
        
        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }
       
        stage('OWASP SCAN') {
            steps {
                dependencyCheck additionalArguments: '', odcInstallation: 'DP-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
    
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs --format table -o fs-report.html ."
            }
        }

	stage('ggshield Scan') {
            steps {
                sh "ggshield scan repo . --api-key=${env.GG_API_KEY}"
            }
        }
	    
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server'){
                   sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=dso_boardgame -Dsonar.projectKey=dso_boardgame\
                          -Dsonar.java.binaries=. '''
                }
            }
        }

	stage('Quality Gate') {
	    steps { 
		    script {
		     waitForQualityGate abortPipeline: false, credentialsId: "sonar-cred"
		    }
	    }
	}
        
        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }
    
        stage('Build & tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t linksrobot/boardgame:latest ."
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                     withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker'){
                        sh "docker push linksrobot/boardgame:latest"
                    }
                }
            }
        }
        
        stage('Deply to Kubernetes') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.83.64:6443') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }
        
        stage('Verify K8 deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.83.64:6443') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
        
        stage('Pipeline Passed') {
            steps {
                echo 'Pipeline Passed'
            }
        }
    }
}
