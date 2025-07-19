pipeline {
    // agent none

    triggers {

        githubPush()
    }

    environment {
        mydockerimage = "harbor.registry.local/jenkins/mylocalimage"
    }

    stages {
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
    stage('Deploy via Ansible Node') {
    agent { label 'master' }
    steps {
        withCredentials([sshUserPrivateKey(
            credentialsId: 'ansible-ssh-key',
            keyFileVariable: 'ANSIBLE_KEY'
        )]) {
            sh """
            ssh -i "/var/lib/jenkins/keys/id_rsa" -o StrictHostKeyChecking=no vagrant@192.168.56.210 '
                if [ ! -d /home/vagrant/project ]; then
                    sudo -u vagrant mkdir -p /home/vagrant/project
                    sudo -u vagrant git clone --single-branch --branch develop https://github.com/dipen674/Ansible_Project.git /home/vagrant/project
                else
                    cd /home/vagrant/project && sudo -u vagrant git pull
                fi
                source /home/vagrant/myenv/bin/activate
                cd /home/vagrant/project/ansible &&
                sudo -u vagrant ansible-playbook deploy-playbook.yaml -i inventory.ini -e "build_number=${BUILD_NUMBER}"
            '
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

                    Best Regards,  
                    DevOps Team
                    """
                }   
        }       
}