pipeline {
  agent any

  environment {
    // Workspace URL that matches the bundle target host
    DATABRICKS_HOST       = 'https://adb-7405617499680447.7.azuredatabricks.net/'
    DATABRICKS_AUTH_TYPE  = 'oauth-m2m'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        powershell 'Get-ChildItem -Force'
      }
    }

    stage('Install Databricks CLI (Go)') {
    steps {
        powershell '''
            $ErrorActionPreference = "Stop"

            Write-Host "Downloading Databricks Go CLI..."

            $cliUrl = "https://github.com/databricks/cli/releases/latest/download/databricks_windows_amd64.zip"
            $zipPath = "$env:WORKSPACE\\databricks_cli.zip"
            $extractPath = "$env:WORKSPACE\\databricks_cli"

            Invoke-WebRequest -Uri $cliUrl -OutFile $zipPath

            Expand-Archive -Path $zipPath -DestinationPath $extractPath -Force

            $exePath = Join-Path $extractPath "databricks.exe"

            if (-Not (Test-Path $exePath)) {
                Write-Error "Databricks CLI not found after extraction!"
            }

            # Add CLI to PATH for this job
            $env:Path = "$env:Path;$extractPath"
            [System.Environment]::SetEnvironmentVariable("PATH", $env:Path, "Process")

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
          powershell '''
            $ErrorActionPreference = "Stop"
            [System.Environment]::SetEnvironmentVariable("DATABRICKS_HOST", $env:DATABRICKS_HOST, "Process")
            [System.Environment]::SetEnvironmentVariable("DATABRICKS_AUTH_TYPE", $env:DATABRICKS_AUTH_TYPE, "Process")
            [System.Environment]::SetEnvironmentVariable("DATABRICKS_CLIENT_ID", $env:DATABRICKS_CLIENT_ID, "Process")
            [System.Environment]::SetEnvironmentVariable("DATABRICKS_CLIENT_SECRET", $env:DATABRICKS_CLIENT_SECRET, "Process")
            Write-Host "Configured Databricks OAuth M2M (Service Principal)."
          '''
        }
      }
    }

    stage('Validate Bundle') {
      steps {
        powershell '''
          $ErrorActionPreference = "Stop"
          if (!(Test-Path "databricks.yml")) {
            Write-Error "databricks.yml not found at workspace root."
          }
          databricks bundle validate -t production
        '''
      }
    }

    stage('Deploy Bundle') {
      steps {
        powershell '''
          $ErrorActionPreference = "Stop"
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