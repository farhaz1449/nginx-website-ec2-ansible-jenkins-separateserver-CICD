pipeline {
    agent any

    environment {
        ANSIBLE_SERVER_IP = 'ANSIBLE_CONTROL_NODE_IP' // e.g. 10.0.1.10
        ANSIBLE_USER      = 'ansible'                 // or ubuntu, etc.
    }

    stages {
        stage('Deploy via Ansible Server') {
            steps {
                sshagent(credentials: ['ansible-server-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no "$ANSIBLE_USER@$ANSIBLE_SERVER_IP" '
                            # Clone or update repo on Ansible server
                            if [ ! -d ~/nginx-website-ec2-ansible-jenkins-CICD ]; then
                                git clone https://github.com/farhaz1449/nginx-website-ec2-ansible-jenkins-CICD.git
                            else
                                cd ~/nginx-website-ec2-ansible-jenkins-CICD && git pull
                            fi

                            # Run Ansible from that repo against remote web servers
                            cd ~/nginx-website-ec2-ansible-jenkins-CICD
                            ansible-playbook web-playbook.yml
                        '
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline execution successful.'
        }
        failure {
            echo 'Pipeline execution failed.'
        }
    }
}
