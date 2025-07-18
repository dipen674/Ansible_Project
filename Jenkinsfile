pipeline {
    agent none

    environment {
        mydockerimage = "harbor.registry.local/jenkins/mylocalimage"
    }

    stages {
        stage('Write Frontend .env in production environment') {
            agent { label 'production' }
            steps {
                writeFile file: './FrontEnd/.env', text: "REACT_APP_API_URL=http://192.168.56.212:5000"
                sh "cat ./FrontEnd/.env"
            }
        }
         stage('Build docker image') {
            agent {label "production"}
            steps {
                echo "Building docker images"
                sh "docker image build -t ${mydockerimage}:frontend_${BUILD_NUMBER} ./FrontEnd"
                sh "docker image build -t ${mydockerimage}:backend_${BUILD_NUMBER} ./BackEnd"
            }
        }
         stage('Image scanning with trivy') {
            agent {label "production"}
            steps {
                echo "Scanning image vulneriblity"
                sh "trivy image ${mydockerimage}:frontend_${BUILD_NUMBER} > trivy_frontend_report.txt"
                sh "trivy image ${mydockerimage}:backend_${BUILD_NUMBER} > trivy_backend_report.txt"
        }
        }
         stage('Pushing docker image to harbor registry') {
            agent {label "production"}
            steps {
                echo "pushing image to docker hub registry"
                withDockerRegistry ([credentialsId: 'Harborregistrycredentials', url: 'https://harbor.registry.local']) {
                    sh '''
                    docker push ${mydockerimage}:frontend_${BUILD_NUMBER}
                    docker push ${mydockerimage}:backend_${BUILD_NUMBER}
                    '''
                }
            }
        }
        stage('Deploy Application') {
            steps {
                dir('ansible') {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'deployment-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh """
                        ansible-playbook deploy-playbook.yml \
                          -e "build_number=${BUILD_NUMBER}" \
                          --private-key ${SSH_KEY}
                        """
                    }
                }
            }
        }
    }
    post {
            always {
            node('deployment') {
                script {
                    sh '''
                    echo "Removing dangling images..."
                    docker image prune -a -f
                    '''
                }
                // cleanWs()
            }
            node('production') {
                script {
                    sh '''
                    echo "Removing dangling images..."
                    docker image prune -a -f
                    '''
                }
                cleanWs()
            }
        }

        success {
                    node('master'){
                    mail bcc: 'dipakbhatt363@gmail.com',
                    to: 'bhattadeependra05@gmail.com',
                    cc: 'bhattad625@gmail.com',
                    from: 'bhattad625@gmail.com',
                    replyTo: '',
                    subject: 'BUILD SUCCESS NOTIFICATION',
                    body: """Hi Team,

                        Build #$BUILD_NUMBER is successful. Please review the build details at:
                        $BUILD_URL

                        Regards,  
                        DevOps Team"""
                    }
                }    
            failure {
                node("master"){
                mail to: 'bhattadeependra05@gmail.com',
                cc: 'dipakbhatt363@gmail.com',
                bcc: '',
                from: 'bhattad625@gmail.com',
                replyTo: 'bhattadeependra05@gmail.com',
                subject: 'BUILD FAILED NOTIFICATION',
                body: """Hi Team,

                    Build #$BUILD_NUMBER is unsuccessful.  
                    Please go through the following URL and verify the details:  
                    $BUILD_URL

                    Regards,  
                    DevOps Team
                    """
                }
            }       
    }
}