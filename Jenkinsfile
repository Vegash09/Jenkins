pipeline {
    agent any
    
    environment {
        DATABRICKS_BUNDLE_ENV = 'production'
        DATABRICKS_CLI_PATH = "${WORKSPACE}/.databricks-cli"
        DATABRICKS_CLI_VERSION = 'v0.234.0'
    }
    
    options {
        skipDefaultCheckout(false)
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
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
                    echo 'Setting up Databricks CLI...'
                    sh '''#!/bin/bash
                        set -euo pipefail
                        
                        CLI_PATH="${DATABRICKS_CLI_PATH}"
                        CLI_EXE="${CLI_PATH}/databricks"
                        
                        # Create directory if it doesn't exist
                        mkdir -p "${CLI_PATH}"
                        echo "CLI directory: ${CLI_PATH}"
                        
                        # Check if CLI is already installed
                        if [ -f "${CLI_EXE}" ]; then
                            echo "Databricks CLI already installed"
                            "${CLI_EXE}" version
                            exit 0
                        fi
                        
                        echo "Installing Databricks CLI ${DATABRICKS_CLI_VERSION}..."
                        
                        # Download URL
                        VERSION="${DATABRICKS_CLI_VERSION#v}"
                        DOWNLOAD_URL="https://github.com/databricks/cli/releases/download/${DATABRICKS_CLI_VERSION}/databricks_cli_${VERSION}_linux_amd64.zip"
                        ZIP_FILE="/tmp/databricks_cli_${BUILD_NUMBER}.zip"
                        
                        echo "Downloading from: ${DOWNLOAD_URL}"
                        curl -fsSL "${DOWNLOAD_URL}" -o "${ZIP_FILE}"
                        
                        # Extract
                        echo "Extracting CLI..."
                        unzip -q -o "${ZIP_FILE}" -d "${CLI_PATH}"
                        
                        # Make executable
                        chmod +x "${CLI_EXE}"
                        
                        # Verify installation
                        if [ ! -f "${CLI_EXE}" ]; then
                            echo "ERROR: Installation failed - databricks executable not found"
                            ls -la "${CLI_PATH}"
                            exit 1
                        fi
                        
                        echo "Installation successful!"
                        "${CLI_EXE}" version
                        
                        # Cleanup
                        rm -f "${ZIP_FILE}"
                    '''
                }
            }
        }
        
        stage('Configure Authentication') {
            steps {
                script {
                    echo 'Configuring Databricks authentication...'
                    withCredentials([
                        string(credentialsId: 'databricks-host', variable: 'DATABRICKS_HOST'),
                        string(credentialsId: 'databricks-client-id', variable: 'DATABRICKS_CLIENT_ID'),
                        string(credentialsId: 'databricks-client-secret', variable: 'DATABRICKS_CLIENT_SECRET')
                    ]) {
                        // Validate credentials are set
                        sh '''#!/bin/bash
                            set -euo pipefail
                            
                            if [ -z "${DATABRICKS_HOST}" ] || [ -z "${DATABRICKS_CLIENT_ID}" ] || [ -z "${DATABRICKS_CLIENT_SECRET}" ]; then
                                echo "ERROR: Missing required Databricks credentials"
                                exit 1
                            fi
                            
                            echo "Databricks Host: ${DATABRICKS_HOST}"
                            echo "Client ID: ${DATABRICKS_CLIENT_ID:0:8}..."
                            echo "Authentication configured successfully"
                        '''
                        
                        // Export credentials for subsequent stages
                        env.DATABRICKS_HOST = DATABRICKS_HOST
                        env.DATABRICKS_CLIENT_ID = DATABRICKS_CLIENT_ID
                        env.DATABRICKS_CLIENT_SECRET = DATABRICKS_CLIENT_SECRET
                    }
                }
            }
        }
        
        stage('Prepare Databricks Notebooks') {
            steps {
                script {
                    echo 'Ensuring Python files have Databricks notebook headers...'
                    sh '''#!/bin/bash
                        set -euo pipefail
                        
                        HEADER="# Databricks notebook source"
                        PROCESSED=0
                        SKIPPED=0
                        
                        echo "Scanning for Python files..."
                        
                        # Process all .py files
                        while IFS= read -r -d '' file; do
                            echo "Checking: ${file}"
                            
                            # Check if header already exists
                            if head -n 1 "${file}" | grep -qF "# Databricks notebook source"; then
                                echo "  ✓ Already has header"
                                ((SKIPPED++))
                            else
                                # Add header
                                echo "${HEADER}" | cat - "${file}" > "${file}.tmp"
                                mv "${file}.tmp" "${file}"
                                echo "  + Added header"
                                ((PROCESSED++))
                            fi
                        done < <(find "${WORKSPACE}" -type f -name "*.py" -print0)
                        
                        echo ""
                        echo "Summary: Processed=${PROCESSED}, Skipped=${SKIPPED}"
                    '''
                }
            }
        }
        
        stage('Validate Bundle') {
            steps {
                script {
                    echo 'Validating Databricks asset bundle configuration...'
                    withCredentials([
                        string(credentialsId: 'databricks-host', variable: 'DATABRICKS_HOST'),
                        string(credentialsId: 'databricks-client-id', variable: 'DATABRICKS_CLIENT_ID'),
                        string(credentialsId: 'databricks-client-secret', variable: 'DATABRICKS_CLIENT_SECRET')
                    ]) {
                        sh """#!/bin/bash
                            set -euo pipefail
                            
                            export PATH=\${PATH}:${DATABRICKS_CLI_PATH}
                            export DATABRICKS_HOST=${DATABRICKS_HOST}
                            export DATABRICKS_CLIENT_ID=${DATABRICKS_CLIENT_ID}
                            export DATABRICKS_CLIENT_SECRET=${DATABRICKS_CLIENT_SECRET}
                            
                            echo "========================================"
                            echo "Databricks Bundle Validation"
                            echo "========================================"
                            echo "Environment: ${DATABRICKS_BUNDLE_ENV}"
                            echo "Host: ${DATABRICKS_HOST}"
                            echo ""
                            
                            databricks bundle validate --environment ${DATABRICKS_BUNDLE_ENV}
                            
                            echo ""
                            echo "✓ Validation successful"
                        """
                    }
                }
            }
        }
        
        stage('Deploy Bundle') {
            when {
                expression { 
                    currentBuild.result == null || currentBuild.result == 'SUCCESS' 
                }
            }
            steps {
                script {
                    echo 'Deploying Databricks asset bundle...'
                    withCredentials([
                        string(credentialsId: 'databricks-host', variable: 'DATABRICKS_HOST'),
                        string(credentialsId: 'databricks-client-id', variable: 'DATABRICKS_CLIENT_ID'),
                        string(credentialsId: 'databricks-client-secret', variable: 'DATABRICKS_CLIENT_SECRET')
                    ]) {
                        sh """#!/bin/bash
                            set -euo pipefail
                            
                            export PATH=\${PATH}:${DATABRICKS_CLI_PATH}
                            export DATABRICKS_HOST=${DATABRICKS_HOST}
                            export DATABRICKS_CLIENT_ID=${DATABRICKS_CLIENT_ID}
                            export DATABRICKS_CLIENT_SECRET=${DATABRICKS_CLIENT_SECRET}
                            
                            echo "========================================"
                            echo "Databricks Bundle Deployment"
                            echo "========================================"
                            echo "Environment: ${DATABRICKS_BUNDLE_ENV}"
                            echo "Host: ${DATABRICKS_HOST}"
                            echo ""
                            
                            databricks bundle deploy --environment ${DATABRICKS_BUNDLE_ENV}
                            
                            echo ""
                            echo "Deployment successful"
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo '=========================================='
            echo 'Pipeline completed successfully!'
            echo '=========================================='
        }
        failure {
            echo '=========================================='
            echo 'Pipeline failed!'
            echo '=========================================='
        }
        always {
            echo 'Cleaning up workspace...'
            cleanWs(
                deleteDirs: true,
                disableDeferredWipeout: true,
                notFailBuild: true
            )
        }
    }
}