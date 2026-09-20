pipeline {
    agent any
    environment {
        COMPOSE_PROJECT = "gestion-caisse"
        SONAR_TOKEN = credentials('sonarqube-token')
    }
    stages {
        stage('Checkout') {
            steps {
                echo 'Recuperation du code source...'
                checkout scm
            }
        }
        stage('SonarQube Analysis') {
            steps {
                echo 'Analyse SonarQube du backend...'
                dir('backend/GestionCaisse') {
                    sh 'chmod +x mvnw'
                    sh './mvnw compile -DskipTests'
                    sh """
                        sonar-scanner \
                        -Dsonar.projectKey=gestion-caisse \
                        -Dsonar.sources=src \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.host.url=http://192.168.56.10:9000 \
                        -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }
        stage('Quality Gate') {
            steps {
                dir('backend/GestionCaisse') {
                    script {
                        writeFile file: 'check_status.sh', text: '''#!/bin/bash
TOKEN=$1
URL=$2
curl -s -u "${TOKEN}:" "${URL}" | grep -o '"status":"[A-Z]*"' | head -1 | cut -d'"' -f4
'''
                        sh 'chmod +x check_status.sh'

                        def taskId = sh(
                            script: "grep ceTaskId .scannerwork/report-task.txt | cut -d= -f2",
                            returnStdout: true
                        ).trim()
                        echo "Task ID SonarQube : ${taskId}"

                        def taskStatus = ""
                        timeout(time: 2, unit: 'MINUTES') {
                            waitUntil {
                                taskStatus = sh(
                                    script: "./check_status.sh '${SONAR_TOKEN}' 'http://192.168.56.10:9000/api/ce/task?id=${taskId}'",
                                    returnStdout: true
                                ).trim()
                                echo "Statut de la tache : ${taskStatus}"
                                return taskStatus == "SUCCESS" || taskStatus == "FAILED" || taskStatus == "CANCELED"
                            }
                        }
                        if (taskStatus != "SUCCESS") {
                            error "Analyse SonarQube echouee (statut : ${taskStatus})"
                        }

                        def qgStatus = sh(
                            script: "./check_status.sh '${SONAR_TOKEN}' 'http://192.168.56.10:9000/api/qualitygates/project_status?projectKey=gestion-caisse'",
                            returnStdout: true
                        ).trim()
                        echo "Statut Quality Gate : ${qgStatus}"
                        if (qgStatus != "OK") {
                            error "Quality Gate echouee : ${qgStatus}"
                        }
                    }
                }
            }
        }        stage('OWASP Dependency-Check') {
            environment {
                NVD_API_KEY = credentials('nvd-api-key')
            }
            steps {
                echo 'Analyse des vulnerabilites des dependances...'
                retry(3) {
                    dependencyCheck additionalArguments: '''
                        --scan backend/GestionCaisse
                        --scan frontend
                        --format HTML
                        --format XML
                        --project gestion-caisse
                        --nvdApiKey ${NVD_API_KEY}
                    ''', odcInstallation: 'OWASP-DC'
                }
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }
        stage('Build Backend Image') {
            steps {
                echo 'Construction de l\'image Docker du backend...'
                retry(3) {
                    sh 'docker compose -f ${WORKSPACE}/docker-compose.yml build backend'
                }
            }
        }
        stage('Build Frontend Image') {
            steps {
                echo 'Construction de l\'image Docker du frontend...'
                retry(3) {
                    sh 'docker compose -f ${WORKSPACE}/docker-compose.yml build frontend'
                }
            }
        }
           stage('Trivy Scan') {
    steps {
        echo 'Scan de securite des images Docker avec Trivy...'
        sh 'trivy image --timeout 15m --severity CRITICAL --exit-code 1 --format table gestion-caisse-pipeline-backend:latest'
        sh 'trivy image --timeout 15m --severity CRITICAL --exit-code 1 --format table gestion-caisse-pipeline-frontend:latest'
    }
}
     stage('Push to Harbor') {
          steps {
                echo 'Push des images validees vers Harbor...'
                withCredentials([usernamePassword(credentialsId: 'harbor-credentials', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                  sh '''
                echo "$HARBOR_PASS" | docker login 192.168.56.10:8082 -u "$HARBOR_USER" --password-stdin
                docker tag gestion-caisse-pipeline-backend:latest 192.168.56.10:8082/gestion-caisse/backend:latest
                docker tag gestion-caisse-pipeline-frontend:latest 192.168.56.10:8082/gestion-caisse/frontend:latest
                docker push 192.168.56.10:8082/gestion-caisse/backend:latest
                docker push 192.168.56.10:8082/gestion-caisse/frontend:latest
            '''
              }
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploiement des conteneurs...'
                sh 'docker compose -f ${WORKSPACE}/docker-compose.yml up -d postgres backend'
            }
        }
        stage('Health Check') {
            steps {
                echo 'Verification que le backend repond...'
                sh 'sleep 10 && curl -I http://192.168.56.10:8081 || true'
            }
        }
    }
    post {
        success {
            echo 'Pipeline termine avec succes !'
        }
        failure {
            echo 'Le pipeline a echoue.'
        }
    }
}
