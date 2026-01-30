pipeline {
  agent any

  environment {
    DATABRICKS_HOST      = 'https://adb-7405617499680447.7.azuredatabricks.net/'
    DATABRICKS_AUTH_TYPE = 'oauth-m2m'
    PATH = "${env.PATH}:${env.HOME}/.databricks/bin"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'ls -la'
      }
    }

    stage('Install Databricks CLI') {
      steps {
        sh '''
          set -e

          if ! command -v databricks >/dev/null 2>&1; then
            echo "Installing Databricks CLI..."
            curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
          else
            echo "Databricks CLI already installed"
          fi

          databricks version
        '''
      }
    }

    stage('Configure Databricks Auth (Service Principal)') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'databricks-sp-cred',
            usernameVariable: 'DATABRICKS_CLIENT_ID',
            passwordVariable: 'DATABRICKS_CLIENT_SECRET'
          )
        ]) {
          sh '''
            set -e

            export DATABRICKS_HOST=$DATABRICKS_HOST
            export DATABRICKS_AUTH_TYPE=$DATABRICKS_AUTH_TYPE
            export DATABRICKS_CLIENT_ID=$DATABRICKS_CLIENT_ID
            export DATABRICKS_CLIENT_SECRET=$DATABRICKS_CLIENT_SECRET

            echo "Databricks Service Principal configured"
          '''
        }
      }
    }

    stage('Validate Bundle') {
      steps {
        sh '''
          set -e

          if [ ! -f databricks.yml ]; then
            echo "❌ databricks.yml not found"
            exit 1
          fi

          databricks bundle validate -t production
        '''
      }
    }

    stage('Deploy Bundle') {
      steps {
        sh '''
          set -e
          databricks bundle deploy -t production
        '''
      }
    }
  }

  post {
    success { echo '✅ Pipeline completed successfully!' }
    failure { echo '❌ Pipeline failed!' }
    always  { cleanWs() }
  }
}
