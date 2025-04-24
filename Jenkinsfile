pipeline {
    agent any

    environment {
        TARGET_GROUP_ARN = 'arn:aws:elasticloadbalancing:il-central-1:314525640319:targetgroup/ofirANDbatel/aa442a8cd866031c'
        BLUE_INSTANCE_ID = 'i-079644fc07481c698'
        GREEN_INSTANCE_ID = 'i-0d2e1ca8cdc3290e7'
        AWS_REGION = 'il-central-1'
    }

    parameters {
        string(name: 'ANSIBLE_PLAYBOOK', defaultValue: 'nginx.yml', description: 'Ansible playbook to run')
        string(name: 'NGINX_PORT', defaultValue: '6789', description: 'Port for Nginx')
        string(name: 'NGINX_STRING', defaultValue: '', description: 'String to configure in Nginx')
    }

    stages {
        stage('Prepare Environment') {
            steps {
                echo "Preparing environment with playbook: ${params.ANSIBLE_PLAYBOOK}"
                echo "NGINX_PORT=${params.NGINX_PORT}, NGINX_STRING=${params.NGINX_STRING}"
            }
        }

        stage('Deploy to BLUE') {
            steps {
                withCredentials([
                    string(credentialsId: 'YOUR_KEY', variable: 'YOUR_KEY'),
                    string(credentialsId: 'YOUR_SECRET', variable: 'YOUR_SECRET'),
                    file(credentialsId: 'KEY-PAIR', variable: 'KEYPAIR_PATH')
                ]) {
                    sh """
                        set -e

                        echo "Generating Ansible inventory and config..."
                        cat <<EOF > hosts
[BLUE]
51.17.22.0

[GREEN]
51.17.79.115

[BLUE:vars]
ansible_user=ubuntu

[GREEN:vars]
ansible_user=ubuntu
EOF

                        echo "======= HOSTS FILE ======="
                        cat hosts
                        echo "=========================="

                        cat <<EOF > ansible.cfg
[defaults]
host_key_checking = False
EOF

                        echo "Configuring AWS CLI..."
                        aws configure set aws_access_key_id \$YOUR_KEY
                        aws configure set aws_secret_access_key \$YOUR_SECRET
                        aws configure set default.region $AWS_REGION

                        echo "Deregistering BLUE instance: $BLUE_INSTANCE_ID"
                        aws elbv2 deregister-targets \
                          --target-group-arn $TARGET_GROUP_ARN \
                          --targets Id=$BLUE_INSTANCE_ID

                        echo "Running Ansible playbook on BLUE..."
                        ansible-playbook -i hosts ${params.ANSIBLE_PLAYBOOK} \
                          -e "nginx_port=${params.NGINX_PORT} NGINX_STRING='${params.NGINX_STRING} from BLUE'" \
                          --limit BLUE \
                          --private-key \$KEYPAIR_PATH

                        echo "Re-registering BLUE instance: $BLUE_INSTANCE_ID"
                        aws elbv2 register-targets \
                          --target-group-arn $TARGET_GROUP_ARN \
                          --targets Id=$BLUE_INSTANCE_ID

                        echo "Waiting for BLUE to become healthy..."
                        for i in {1..30}; do
                          STATUS=\$(aws elbv2 describe-target-health \
                            --target-group-arn $TARGET_GROUP_ARN \
                            --query "TargetHealthDescriptions[?Target.Id=='$BLUE_INSTANCE_ID'].TargetHealth.State" \
                            --output text)

                          echo "Health status: \$STATUS"
                          if [ "\$STATUS" = "healthy" ]; then
                            echo "BLUE instance is healthy!"
                            break
                          fi
                          sleep 10
                        done
                    """
                }
            }
        }

        stage('Deploy to GREEN') {
            steps {
                withCredentials([
                    string(credentialsId: 'YOUR_KEY', variable: 'YOUR_KEY'),
                    string(credentialsId: 'YOUR_SECRET', variable: 'YOUR_SECRET'),
                    file(credentialsId: 'KEY-PAIR', variable: 'KEYPAIR_PATH')
                ]) {
                    sh """
                        set -e

                        echo "Deregistering GREEN instance: $GREEN_INSTANCE_ID"
                        aws elbv2 deregister-targets \
                          --target-group-arn $TARGET_GROUP_ARN \
                          --targets Id=$GREEN_INSTANCE_ID

                        echo "Running Ansible playbook on GREEN..."
                        ansible-playbook -i hosts ${params.ANSIBLE_PLAYBOOK} \
                          -e "nginx_port=${params.NGINX_PORT} NGINX_STRING='${params.NGINX_STRING} from GREEN'" \
                          --limit GREEN \
                          --private-key \$KEYPAIR_PATH

                        echo "Re-registering GREEN instance: $GREEN_INSTANCE_ID"
                        aws elbv2 register-targets \
                          --target-group-arn $TARGET_GROUP_ARN \
                          --targets Id=$GREEN_INSTANCE_ID

                        echo "Waiting for GREEN to become healthy..."
                        for i in {1..30}; do
                          STATUS=\$(aws elbv2 describe-target-health \
                            --target-group-arn $TARGET_GROUP_ARN \
                            --query "TargetHealthDescriptions[?Target.Id=='$GREEN_INSTANCE_ID'].TargetHealth.State" \
                            --output text)

                          echo "Health status: \$STATUS"
                          if [ "\$STATUS" = "healthy" ]; then
                            echo "GREEN instance is healthy!"
                            break
                          fi
                          sleep 10
                        done
                    """
                }
            }
        }
    }
}
