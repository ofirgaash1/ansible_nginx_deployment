pipeline {
    agent any
    environment {
        VAULT_TOKEN = credentials('vault-token-secret-text')
        VM_HOST = '51.84.148.125'
        VM_USER = 'ubuntu'
    }
    stages {
        stage('Get SSH Private Key from Vault') {
            steps {
                script {
                    // Create a directory for storing credentials securely
                    sh 'mkdir -p ~/.ssh_temp && chmod 700 ~/.ssh_temp'
                    
                    // Retrieve private key from Vault using direct curl with basic shell processing
                    withCredentials([string(credentialsId: 'vault-token-secret-text', variable: 'VAULT_TOKEN')]) {
                        sh '''
                            # Get response from Vault 
                            RESPONSE=$(curl --silent --header "X-Vault-Token: $VAULT_TOKEN" --request GET http://vault:8200/v1/secret/aws/privat-key)
                            
                            # Extract the private key using basic shell commands
                            # Locate the beginning of the key and extract to the end
                            echo "$RESPONSE" | sed 's/.*"value":"//' | sed 's/".*//' > ~/.ssh_temp/ssh_key.pem
                            
                            # Fix newlines (replace \\n with actual newlines)
                            sed -i 's/\\\\n/\\n/g' ~/.ssh_temp/ssh_key.pem
                            
                            # Set correct permissions
                            chmod 600 ~/.ssh_temp/ssh_key.pem
                        '''
                    }
                    
                    echo "Private Key retrieved and stored securely"
                    
                    // Test connection to VM
                    sh """
                        ssh -o StrictHostKeyChecking=no -i ~/.ssh_temp/ssh_key.pem ${VM_USER}@${VM_HOST} 'echo "Successfully connected to the virtual machine!"'
                    """
                }
            }
        }
        
        stage('Run Ansible Playbook') {
            steps {
                script {

                    sh '''
                        # Check if playbook exists in workspace
                        if [ -f "playbook-Nginx.yml" ]; then
                            echo "Using playbook from workspace"
                        else
                            echo "ERROR: playbook-Nginx.yml not found in workspace"
                            exit 1
                        fi
                        
                        # Create a simple ansible inventory file for this host
                        echo "[webserver]" > ~/.ssh_temp/inventory
                        echo "${VM_HOST} ansible_user=${VM_USER} ansible_ssh_private_key_file=~/.ssh_temp/ssh_key.pem ansible_ssh_common_args='-o StrictHostKeyChecking=no'" >> ~/.ssh_temp/inventory
                        
                        # Display inventory for debugging
                        cat ~/.ssh_temp/inventory
                        
                        # Run the playbook
                        ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i ~/.ssh_temp/inventory playbook-Nginx.yml
                    '''

                }
            }
        }
        
        stage('Cleanup') {
            steps {
                // Clean up the SSH key after use
                sh 'rm -rf ~/.ssh_temp'
                echo "Credentials cleaned up"
            }
        }
    }
    
    post {
        always {
            // Ensure credentials are always cleaned up, even if the pipeline fails
            sh 'rm -rf ~/.ssh_temp || true'
        }
    }
}