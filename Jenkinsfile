pipeline {
    agent {
        label 'linux' // Use a Linux agent
    }
    
    environment {
        // Hard-coded credentials (NOT RECOMMENDED for production)
        DATABRICKS_CLIENT_ID = 'e1720236-6ef3-47c5-b73e-700d4a171681'
        DATABRICKS_CLIENT_SECRET = 'dosed0e6ab7e4b30bb5ba34f5b3971c88456'
        DATABRICKS_HOST = 'https://adb-7405617499680447.7.azuredatabricks.net/'
        DATABRICKS_BUNDLE_ENV = 'production' // Set your environment name
        BUNDLE_VAR_DATABRICKS_CLIENT_ID = 'e1720236-6ef3-47c5-b73e-700d4a171681'
    }
    
    options {
        skipDefaultCheckout(false)
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub repository...'
                checkout scm
            }
        }
        
        stage('Setup Databricks CLI') {
            steps {
                script {
                    echo 'Installing Databricks CLI...'
                    sh '''
                        # Install Databricks CLI
                        curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
                        
                        # Verify installation
                        databricks --version
                    '''
                }
            }
        }
        
        stage('Ensure Python files are Databricks notebooks') {
            steps {
                script {
                    echo 'Prepending Databricks notebook header to all .py files...'
                    sh '''
                        # Install PowerShell on Ubuntu if not already installed
                        if ! command -v pwsh &> /dev/null; then
                            echo "Installing PowerShell..."
                            apt-get update
                            apt-get install -y wget apt-transport-https software-properties-common
                            wget -q "https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/packages-microsoft-prod.deb"
                            dpkg -i packages-microsoft-prod.deb
                            apt-get update
                            apt-get install -y powershell
                        fi
                        
                        # Use PowerShell to prepend '# Databricks notebook source' to all .py files
                        pwsh -Command "Get-ChildItem -Path '${WORKSPACE}' -Filter '*.py' -Recurse | ForEach-Object { ('# Databricks notebook source`n' + (Get-Content \\$_.FullName -Raw)) | Set-Content \\$_.FullName }"
                    '''
                }
            }
        }
        
        stage('Validate Databricks Bundle') {
            steps {
                script {
                    echo 'Validating Databricks asset bundle...'
                    sh '''
                        databricks bundle validate
                    '''
                }
            }
        }
        
        stage('Deploy Databricks Bundle') {
            when {
                expression { 
                    currentBuild.result == null || currentBuild.result == 'SUCCESS' 
                }
            }
            steps {
                script {
                    echo 'Deploying Databricks asset bundle...'
                    sh '''
                        databricks bundle deploy
                    '''
                }
            }
        }
    }
    
    post {
        failure {
            echo 'Pipeline failed!'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        always {
            cleanWs() // Clean workspace after build
        }
    }
}