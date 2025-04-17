pipeline {
  agent any

  stages {
    stage('Test Vault SSH Key') {
      steps {
        withCredentials([sshUserPrivateKey(
          credentialsId: 'vault-ansible-key',
          keyFileVariable: 'PRIVATE_KEY_PATH',
          usernameVariable: 'SSH_USER'
        )]) {
          sh '''
            echo "=== בדיקת שליפת מפתח פרטי מוולט ==="
            echo "USER: $SSH_USER"
            echo "  PATH : $PRIVATE_KEY_PATH"
            head -n 5 $PRIVATE_KEY_PATH || echo "נכשל בקריאת המפתח"
          '''
        }
      }
    }
  }
}