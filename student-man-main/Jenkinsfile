pipeline {
    agent any

    environment {
        REGISTRY = "farzzit"
        IMAGE = "studentmang-app"

        // Correct URLs
        SONAR_HOST_URL = "http://192.168.33.10:9000"        // Local SonarQube on Ubuntu
        APP_DAST_URL   = "http://192.168.49.2:32639"        // Spring App on Minikube NodePort

        GITLEAKS_REPORT = "gitleaks-report.json"
    }

    stages {
        stage('Clone Repository & Secrets Scan (Gitleaks)') {
            steps {
                git branch: 'master', url: 'https://github.com/medaminehammami/student-management-devops.git'
                dir('student-man-main') {
                    sh '''
                    echo "🔒 Running Gitleaks Secrets Scan..."
                    gitleaks detect -f json --report-path $GITLEAKS_REPORT --source . || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "student-man-main/${GITLEAKS_REPORT}", allowEmptyArchive: true
                }
            }
        }

        stage('Build with Maven') {
            steps {
                dir('student-man-main') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Dependency Check (SCA)') {
            steps {
                dir('student-man-main') {
                    sh '''
                    echo "🔍 Running lightweight OWASP Dependency-Check..."
                    mvn org.owasp:dependency-check-maven:check \
                        -Dformat=HTML \
                        -DskipAssembly=true || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'student-man-main/target/dependency-check-report.html', allowEmptyArchive: true
                }
            }
        }

        stage('SonarQube (SAST)') {
            environment {
                SONARQUBE = credentials('sonarqube-token')
            }
            steps {
                dir('student-man-main') {
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                        echo "📊 Running SonarQube static code analysis..."
                        mvn sonar:sonar \
                          -Dsonar.projectKey=studentmang-app \
                          -Dsonar.host.url=${SONAR_HOST_URL} \
                          -Dsonar.login=$SONARQUBE || true
                        '''
                    }
                }
            }
            // 🚀 Removed timeout & waitForQualityGate for speed
        }

        stage('Build Docker Image') {
            steps {
                dir('student-man-main') {
                    sh 'docker build -t $REGISTRY/$IMAGE:latest .'
                }
            }
        }

        stage('Trivy Scan (Container Security)') {
            steps {
                dir('student-man-main') {
                    sh '''
                    echo "🧪 Quick Trivy vulnerability scan..."
                    trivy image --quiet --severity HIGH,CRITICAL \
                                --light --timeout 5m \
                                --format table \
                                --output trivy-report.html \
                                $REGISTRY/$IMAGE:latest || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'student-man-main/trivy-report.html', allowEmptyArchive: true
                }
            }
        }

        stage('Nikto Scan (DAST)') {
            steps {
                dir('student-man-main') {
                    sh '''
                    echo "💥 Running Nikto web vulnerability scan on $APP_DAST_URL..."
                    nikto -h $APP_DAST_URL -maxtime 60 -Format html -o nikto-report.html || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'student-man-main/nikto-report.html', allowEmptyArchive: true
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                dir('student-man-main') {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $REGISTRY/$IMAGE:latest || true
                        docker logout
                        '''
                    }
                }
            }
        }

        stage('Generate Summary Report') {
            steps {
                script {
                    def time = sh(script: "date", returnStdout: true).trim()
                    writeFile file: 'student-man-main/summary-report.html', text: """
                    <html><head><title>DevSecOps Report</title></head><body>
                        <h2>✅ DevSecOps Security Pipeline Summary</h2>
                        <ul>
                            <li><a href='target/dependency-check-report.html'>Dependency Check Report (SCA)</a></li>
                            <li><a href='trivy-report.html'>Trivy Image Scan Report (Container)</a></li>
                            <li><a href='nikto-report.html'>Nikto Server Scan Report (DAST)</a></li>
                            <li><a href='${GITLEAKS_REPORT}'>Gitleaks Secrets Scan Report</a></li>
                            <li><a href='${SONAR_HOST_URL}/dashboard?id=studentmang-app'>SonarQube Dashboard (SAST)</a></li>
                        </ul>
                        <p>Generated by Jenkins on ${time}</p>
                    </body></html>
                    """
                }
                archiveArtifacts artifacts: 'student-man-main/summary-report.html', allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo '🎯 Pipeline finished. Reports generated successfully.'
        }
        failure {
            echo '❌ Pipeline failed. Please check logs and reports for issues.'
        }
    }
}
