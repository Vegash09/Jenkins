pipeline {
  agent any

  environment {
    // Use the workspace host that matches your bundle target host
    // If your databricks.yml target.host is dbc-... then use that here as well.
    DATABRICKS_HOST      = 'https://adb-7405617499680447.7.azuredatabricks.net/'
    DATABRICKS_AUTH_TYPE = 'oauth-m2m'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'ls -la'
      }
    }

    stage('Install Databricks CLI (Go) - Linux') {
      steps {
        sh '''
          set -euo pipefail
          echo "Downloading Databricks Go CLI for Linux..."

          # Prepare paths in workspace
          CLI_DIR="$WORKSPACE/databricks_cli"
          mkdir -p "$CLI_DIR"

          # Download the latest Linux amd64 tarball and extract
          CLI_TGZ="$WORKSPACE/databricks_linux_amd64.tar.gz"
          curl -fsSL -o "$CLI_TGZ" "https://github.com/databricks/cli/releases/latest/download/databricks_linux_amd64.tar.gz"

          tar -xzf "$CLI_TGZ" -C "$CLI_DIR"

          # Add to PATH for current process
          export PATH="$CLI_DIR:$PATH"
          echo "PATH after adding CLI: $PATH"

          databricks -v
        '''
      }
    }

    stage('Configure Databricks Auth (Service Principal OAuth M2M)') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'databricks-sp-cred',
            usernameVariable: 'DATABRICKS_CLIENT_ID',
            passwordVariable: 'DATABRICKS_CLIENT_SECRET'
          )
        ]) {
          sh '''
            set -euo pipefail
            # Make sure CLI we just installed is on PATH in this shell too
            export PATH="$WORKSPACE/databricks_cli:$PATH"

            # Export env vars for this step and subsequent steps in this stage invocation
            export DATABRICKS_HOST="${DATABRICKS_HOST}"
            export DATABRICKS_AUTH_TYPE="${DATABRICKS_AUTH_TYPE}"
            export DATABRICKS_CLIENT_ID="${DATABRICKS_CLIENT_ID}"
            export DATABRICKS_CLIENT_SECRET="${DATABRICKS_CLIENT_SECRET}"

            echo "Configured Databricks OAuth M2M (Service Principal)."
            echo "CLI version:"; databricks -v
            echo "Auth context:"; databricks auth env
          '''
        }
      }
    }

    stage('Sanity Check (optional)') {
      steps {
        sh '''
          set -euo pipefail
          export PATH="$WORKSPACE/databricks_cli:$PATH"

          if [ ! -f "databricks.yml" ]; then
            echo "databricks.yml not found at the repo root." >&2
            exit 1
          fi

          echo "Showing bundle file:"
          sed -n '1,120p' databricks.yml || true
        '''
      }
    }

    stage('Validate Bundle') {
      steps {
        sh '''
          set -euo pipefail
          export PATH="$WORKSPACE/databricks_cli:$PATH"

          databricks bundle validate -t production
        '''
      }
    }

    stage('Deploy Bundle') {
      steps {
        sh '''
          set -euo pipefail
          export PATH="$WORKSPACE/databricks_cli:$PATH"

          databricks bundle deploy -t production
        '''
      }
    }

    // Optional: run a specific job/pipeline as smoke test after deploy
    // stage('Run Smoke Test') {
    //   steps {
    //     sh '''
    //       set -euo pipefail
    //       export PATH="$WORKSPACE/databricks_cli:$PATH"
    //       databricks bundle run -t production daily_job
    //     '''
    //   }
    // }
  }

  post {
    success { echo '✅ Pipeline completed successfully!' }
    failure { echo '❌ Pipeline failed!' }
    always  { cleanWs() }
  }
}