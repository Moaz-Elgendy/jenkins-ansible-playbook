pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        HOST_IP = '172.17.0.1'
    }

    stages {
        stage('1) apt update') {
            steps {
                sh 'sudo apt-get update -y'
            }
        }

        stage('2) Install Ansible') {
            steps {
                sh 'sudo apt-get install -y ansible'
                sh 'ansible --version'
            }
        }

        stage('3) Ansible dry run (check mode)') {
            steps {
                // --check = dry run, no changes are made on the host
                sh 'ansible-playbook -i inventory.ini --check --diff nginx.yml'
            }
        }

        stage('4) Check Ansible status') {
            steps {
                sh '''
                    set -e
                    if ansible -i inventory.ini nginx_servers -m ping; then
                        echo "Ansible connection OK - proceeding to real run"
                    else
                        echo "ERROR: Ansible cannot reach the host - aborting"
                        exit 1
                    fi
                '''
            }
        }

        stage('5) Apply playbook to real host') {
            steps {
                sh 'ansible-playbook -i inventory.ini nginx.yml'
            }
        }

        stage('6) Verify nginx responds') {
            steps {
                sh '''
                    set -e
                    echo "Waiting for nginx to come up..."
                    for i in $(seq 1 10); do
                        code=$(curl -s -o /dev/null -w "%{http_code}" "http://${HOST_IP}")
                        if echo "$code" | grep -qE "^(200|301|302)$"; then
                            echo "SUCCESS: nginx is reachable at http://${HOST_IP}"
                            exit 0
                        fi
                        sleep 3
                    done
                    echo "ERROR: nginx did not respond on http://${HOST_IP}"
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            sh 'echo "SUCCESS: nginx playbook applied - nginx is installed/updated, enabled, running and verified reachable."'
        }
        failure {
            sh 'echo "ERROR: pipeline failed - playbook was NOT fully applied. Check the stage logs above."'
            error('Pipeline aborted: ansible/nginx failure')
        }
        always {
            echo "Build finished with status: ${currentBuild.currentResult}"
        }
    }
}
