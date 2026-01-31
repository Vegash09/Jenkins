pipeline {
    agent any
    
    environment {
        DATABRICKS_BUNDLE_ENV = 'production'
        DATABRICKS_CLI_PATH = "${WORKSPACE}/.databricks-cli"
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
                    echo 'Downloading and installing Databricks CLI...'
                    sh '''#!/bin/bash
                        set -e
                        
                        CLI_PATH="${DATABRICKS_CLI_PATH}"
                        CLI_EXE="${CLI_PATH}/databricks"
                        
                        # Create directory (no sudo needed - using workspace)
                        if [ ! -d "$CLI_PATH" ]; then
                            mkdir -p "$CLI_PATH"
                            echo "Created directory: $CLI_PATH"
                        fi
                        
                        # Check if already installed
                        if [ -f "$CLI_EXE" ]; then
                            echo "Databricks CLI already exists!"
                            "$CLI_EXE" version
                            exit 0
                        fi
                        
                        echo "Downloading Databricks CLI..."
                        
                        # Use the correct URL format for Linux
                        VERSION="v0.234.0"
                        DOWNLOAD_URL="https://github.com/databricks/cli/releases/download/${VERSION}/databricks_cli_${VERSION#v}_linux_amd64.zip"
                        ZIP_FILE="/tmp/databricks_cli_$$.zip"
                        
                        echo "Download URL: $DOWNLOAD_URL"
                        
                        # Download
                        curl -L "$DOWNLOAD_URL" -o "$ZIP_FILE"
                        echo "Downloaded successfully!"
                        
                        # Extract
                        echo "Extracting..."
                        unzip -o "$ZIP_FILE" -d "$CLI_PATH"
                        
                        # Make executable
                        chmod +x "$CLI_EXE"
                        
                        # Verify extraction
                        if [ -f "$CLI_EXE" ]; then
                            echo "Installation successful!"
                            "$CLI_EXE" version
                        else
                            echo "ERROR: databricks not found after extraction!"
                            echo "Contents of $CLI_PATH:"
                            ls -la "$CLI_PATH"
                            exit 1
                        fi
                        
                        # Cleanup
                        rm -f "$ZIP_FILE"
                    '''
                }
            }
        }
        
        stage('Configure Databricks Authentication') {
            steps {
                script {
                    echo 'Setting up Databricks authentication...'
                    withCredentials([
                        string(credentialsId: 'databricks-client-id', variable: 'DATABRICKS_CLIENT_ID'),
                        string(credentialsId: 'databricks-client-secret', variable: 'DATABRICKS_CLIENT_SECRET'),
                        string(credentialsId: 'databricks-host', variable: 'DATABRICKS_HOST')
                    ]) {
                        sh '''#!/bin/bash
                            set -e
                            
                            echo "Setting Databricks environment variables..."
                            
                            # Verify credentials are set
                            echo "DATABRICKS_HOST: ${DATABRICKS_HOST}"
                            # Show only first 8 chars using bash substring
                            CLIENT_ID_PREVIEW=$(echo ${DATABRICKS_CLIENT_ID} | cut -c1-8)
                            echo "DATABRICKS_CLIENT_ID: ${CLIENT_ID_PREVIEW}..."
                            echo "Authentication configured successfully!"
                        '''
                        
                        // Store credentials in environment for subsequent stages
                        env.DATABRICKS_HOST = DATABRICKS_HOST
                        env.DATABRICKS_CLIENT_ID = DATABRICKS_CLIENT_ID
                        env.DATABRICKS_CLIENT_SECRET = DATABRICKS_CLIENT_SECRET
                        env.BUNDLE_VAR_DATABRICKS_CLIENT_ID = DATABRICKS_CLIENT_ID
                    }
                }
            }
        }
        
        stage('Ensure Python files are Databricks notebooks') {
            steps {
                script {
                    echo 'Prepending Databricks notebook header to all .py files...'
                    sh '''#!/bin/bash
                        HEADER="# Databricks notebook source"
                        PROCESSED_COUNT=0
                        SKIPPED_COUNT=0
                        
                        echo "Scanning for Python files in: ${WORKSPACE}"
                        
                        # Find all .py files
                        find "${WORKSPACE}" -type f -name "*.py" | while read -r file; do
                            echo "Found: $(basename "$file")"
                            
                            # Check if file already has header
                            if head -n 1 "$file" | grep -q "^# Databricks notebook source"; then
                                echo "  Already has header"
                                SKIPPED_COUNT=$((SKIPPED_COUNT + 1))
                            else
                                # Add header
                                echo "$HEADER" | cat - "$file" > "${file}.tmp"
                                mv "${file}.tmp" "$file"
                                echo "  Added header"
                                PROCESSED_COUNT=$((PROCESSED_COUNT + 1))
                            fi
                        done
                        
                        echo ""
                        echo "Processed: $PROCESSED_COUNT | Skipped: $SKIPPED_COUNT"
                    '''
                }
            }
        }
        
        stage('Validate Databricks Bundle') {
            steps {
                script {
                    echo 'Validating Databricks asset bundle...'
                    withCredentials([
                        string(credentialsId: 'databricks-client-id', variable: 'DATABRICKS_CLIENT_ID'),
                        string(credentialsId: 'databricks-client-secret', variable: 'DATABRICKS_CLIENT_SECRET'),
                        string(credentialsId: 'databricks-host', variable: 'DATABRICKS_HOST')
                    ]) {
                        sh """#!/bin/bash
                            set -e
                            export PATH=\$PATH:${DATABRICKS_CLI_PATH}
                            export DATABRICKS_HOST=${DATABRICKS_HOST}
                            export DATABRICKS_CLIENT_ID=${DATABRICKS_CLIENT_ID}
                            export DATABRICKS_CLIENT_SECRET=${DATABRICKS_CLIENT_SECRET}
                            
                            echo ""
                            echo "========================================"
                            echo "Running: databricks bundle validate"
                            echo "========================================"
                            echo ""
                            
                            databricks bundle validate
                            
                            echo ""
                            echo "Validation completed successfully!"
                        """
                    }
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
                    withCredentials([
                        string(credentialsId: 'databricks-client-id', variable: 'DATABRICKS_CLIENT_ID'),
                        string(credentialsId: 'databricks-client-secret', variable: 'DATABRICKS_CLIENT_SECRET'),
                        string(credentialsId: 'databricks-host', variable: 'DATABRICKS_HOST')
                    ]) {
                        sh """#!/bin/bash
                            set -e
                            export PATH=\$PATH:${DATABRICKS_CLI_PATH}
                            export DATABRICKS_HOST=${DATABRICKS_HOST}
                            export DATABRICKS_CLIENT_ID=${DATABRICKS_CLIENT_ID}
                            export DATABRICKS_CLIENT_SECRET=${DATABRICKS_CLIENT_SECRET}
                            export BUNDLE_VAR_DATABRICKS_CLIENT_ID=${DATABRICKS_CLIENT_ID}
                            
                            echo ""
                            echo "========================================"
                            echo "Running: databricks bundle deploy"
                            echo "========================================"
                            echo ""
                            
                            databricks bundle deploy
                            
                            echo ""
                            echo "Deployment completed successfully!"
                        """
                    }
                }
            }
        }
    }
    
    post {
        failure {
            echo '=========================================='
            echo 'Pipeline FAILED!'
            echo '=========================================='
        }
        success {
            echo '=========================================='
            echo 'Pipeline completed SUCCESSFULLY!'
            echo '=========================================='
        }
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
    }
}
