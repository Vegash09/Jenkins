pipeline {
    agent any
    
    parameters {
        choice(
            name: 'DEPLOYMENT_MODE',
            choices: ['Validate Only', 'Validate and Deploy'],
            description: 'Choose the deployment mode'
        )
        booleanParam(
            name: 'REQUIRE_APPROVAL',
            defaultValue: true,
            description: 'Require manual approval before deployment?'
        )
    }
    
    environment {
        DATABRICKS_BUNDLE_ENV = 'production'
        APPROVER_EMAIL = 'vegash.p@diggibyte.com'
    }
    
    options {
        skipDefaultCheckout(false)
        timeout(time: 1, unit: 'HOURS')
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub repository...'
                checkout scm
            }
        }
        
        stage('Display Build Parameters') {
            steps {
                script {
                    echo '=========================================='
                    echo 'BUILD PARAMETERS'
                    echo '=========================================='
                    echo "Deployment Mode: ${params.DEPLOYMENT_MODE}"
                    echo "Require Approval: ${params.REQUIRE_APPROVAL}"
                    echo "Approver: ${env.APPROVER_EMAIL}"
                    echo '=========================================='
                }
            }
        }
        
        stage('Setup Databricks CLI') {
            steps {
                script {
                    echo 'Downloading and installing Databricks CLI...'
                    powershell '''
                        $ErrorActionPreference = "Stop"
                        
                        $cliPath = "C:\\databricks-cli"
                        $cliExe = "$cliPath\\databricks.exe"
                        
                        # Create directory
                        if (!(Test-Path $cliPath)) {
                            New-Item -ItemType Directory -Path $cliPath -Force | Out-Null
                            Write-Host "Created directory: $cliPath"
                        }
                        
                        # Check if already installed
                        if (Test-Path $cliExe) {
                            Write-Host "Databricks CLI already exists!"
                            & $cliExe version
                            exit 0
                        }
                        
                        Write-Host "Downloading Databricks CLI..."
                        
                        # Use the correct URL format
                        $version = "v0.234.0"  # Using a stable version
                        $downloadUrl = "https://github.com/databricks/cli/releases/download/$version/databricks_cli_${version}_windows_amd64.zip"
                        $zipFile = "$env:TEMP\\databricks_cli.zip"
                        
                        Write-Host "Download URL: $downloadUrl"
                        
                        try {
                            # Download with progress
                            $ProgressPreference = 'SilentlyContinue'
                            Invoke-WebRequest -Uri $downloadUrl -OutFile $zipFile -UseBasicParsing
                            Write-Host "Downloaded successfully!"
                            
                            # Extract
                            Write-Host "Extracting..."
                            Expand-Archive -Path $zipFile -DestinationPath $cliPath -Force
                            
                            # Verify extraction
                            if (Test-Path $cliExe) {
                                Write-Host "Installation successful!"
                                & $cliExe version
                            } else {
                                Write-Host "ERROR: databricks.exe not found after extraction!"
                                Write-Host "Contents of $cliPath :"
                                Get-ChildItem $cliPath | Format-Table Name
                                exit 1
                            }
                            
                            # Cleanup
                            Remove-Item $zipFile -Force -ErrorAction SilentlyContinue
                            
                        } catch {
                            Write-Host "ERROR: $_"
                            Write-Host "Failed URL: $downloadUrl"
                            
                            # Try alternate method - download latest directly
                            Write-Host "`nTrying alternate download method..."
                            
                            try {
                                # Download databricks.exe directly without version
                                $altUrl = "https://github.com/databricks/cli/releases/download/v0.234.0/databricks_cli_0.234.0_windows_amd64.zip"
                                Invoke-WebRequest -Uri $altUrl -OutFile $zipFile -UseBasicParsing
                                Expand-Archive -Path $zipFile -DestinationPath $cliPath -Force
                                
                                if (Test-Path $cliExe) {
                                    Write-Host "Alternate download successful!"
                                    & $cliExe version
                                } else {
                                    throw "Still could not find databricks.exe"
                                }
                            } catch {
                                Write-Host "ERROR: Alternate method also failed: $_"
                                exit 1
                            }
                        }
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
                        powershell '''
                            $ErrorActionPreference = "Stop"
                            
                            Write-Host "Setting Databricks environment variables..."
                            
                            # Set environment variables for this session
                            $env:DATABRICKS_HOST = $env:DATABRICKS_HOST
                            $env:DATABRICKS_CLIENT_ID = $env:DATABRICKS_CLIENT_ID
                            $env:DATABRICKS_CLIENT_SECRET = $env:DATABRICKS_CLIENT_SECRET
                            
                            Write-Host "DATABRICKS_HOST: $env:DATABRICKS_HOST"
                            Write-Host "DATABRICKS_CLIENT_ID: $($env:DATABRICKS_CLIENT_ID.Substring(0,8))..." # Show only first 8 chars
                            Write-Host "Authentication configured successfully!"
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
                    powershell '''
                        $header = "# Databricks notebook source"
                        $processedCount = 0
                        $skippedCount = 0
                        
                        Write-Host "Scanning for Python files in: $env:WORKSPACE"
                        
                        Get-ChildItem -Path $env:WORKSPACE -Filter "*.py" -Recurse | ForEach-Object {
                            $file = $_.FullName
                            Write-Host "Found: $($_.Name)"
                            
                            try {
                                $content = Get-Content $file -Raw -ErrorAction Stop
                                
                                if ($content -and $content -notmatch "^# Databricks notebook source") {
                                    $newContent = "$header`n$content"
                                    Set-Content -Path $file -Value $newContent -NoNewline
                                    Write-Host "  Added header" -ForegroundColor Green
                                    $processedCount++
                                } else {
                                    Write-Host "  Already has header" -ForegroundColor Yellow
                                    $skippedCount++
                                }
                            } catch {
                                Write-Host "  Error: $_" -ForegroundColor Red
                            }
                        }
                        
                        Write-Host "`nProcessed: $processedCount | Skipped: $skippedCount"
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
                        bat """
                            set PATH=%PATH%;C:\\databricks-cli
                            set DATABRICKS_HOST=${DATABRICKS_HOST}
                            set DATABRICKS_CLIENT_ID=${DATABRICKS_CLIENT_ID}
                            set DATABRICKS_CLIENT_SECRET=${DATABRICKS_CLIENT_SECRET}
                            
                            echo.
                            echo ========================================
                            echo Running: databricks bundle validate
                            echo ========================================
                            echo.
                            
                            databricks bundle validate
                            
                            echo.
                            echo Validation completed successfully!
                        """
                    }
                }
            }
        }
        
        stage('Approval Gate') {
            when {
                expression {
                    // Only require approval if "Validate and Deploy" is selected and approval is enabled
                    params.DEPLOYMENT_MODE == 'Validate and Deploy' &&
                    params.REQUIRE_APPROVAL == true
                }
            }
            steps {
                script {
                    echo '=========================================='
                    echo 'WAITING FOR DEPLOYMENT APPROVAL'
                    echo '=========================================='
                    echo "Approval required from: ${env.APPROVER_EMAIL}"
                    echo "Build URL: ${env.BUILD_URL}"
                    echo '=========================================='
                    
                    try {
                        timeout(time: 30, unit: 'MINUTES') {
                            def approvalInput = input(
                                id: 'DeployApproval',
                                message: "Deployment approval required from ${env.APPROVER_EMAIL}. Do you approve deployment to Production?",
                                parameters: [
                                    choice(
                                        name: 'APPROVE_DEPLOY',
                                        choices: ['Approve', 'Reject'],
                                        description: 'Approve or Reject the deployment'
                                    ),
                                    text(
                                        name: 'APPROVAL_COMMENT',
                                        defaultValue: '',
                                        description: 'Optional: Add a comment about this deployment'
                                    )
                                ],
                                submitter: 'vegash.p@diggibyte.com',
                                submitterParameter: 'APPROVED_BY'
                            )
                            
                            if (approvalInput.APPROVE_DEPLOY == 'Reject') {
                                error("Deployment rejected by ${approvalInput.APPROVED_BY}")
                            }
                            
                            echo '=========================================='
                            echo "✓ Deployment APPROVED by: ${approvalInput.APPROVED_BY}"
                            if (approvalInput.APPROVAL_COMMENT) {
                                echo "Comment: ${approvalInput.APPROVAL_COMMENT}"
                            }
                            echo '=========================================='
                        }
                    } catch (err) {
                        echo '=========================================='
                        echo "✗ Deployment REJECTED or TIMEOUT"
                        echo "Reason: ${err.message}"
                        echo '=========================================='
                        error("Deployment aborted: ${err.message}")
                    }
                }
            }
        }
        
        stage('Deploy Databricks Bundle') {
            when {
                expression { 
                    // Only deploy if "Validate and Deploy" is selected
                    params.DEPLOYMENT_MODE == 'Validate and Deploy' &&
                    (currentBuild.result == null || currentBuild.result == 'SUCCESS')
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
                        bat """
                            set PATH=%PATH%;C:\\databricks-cli
                            set DATABRICKS_HOST=${DATABRICKS_HOST}
                            set DATABRICKS_CLIENT_ID=${DATABRICKS_CLIENT_ID}
                            set DATABRICKS_CLIENT_SECRET=${DATABRICKS_CLIENT_SECRET}
                            set BUNDLE_VAR_DATABRICKS_CLIENT_ID=${DATABRICKS_CLIENT_ID}
                            
                            echo.
                            echo ========================================
                            echo Running: databricks bundle deploy
                            echo ========================================
                            echo.
                            
                            databricks bundle deploy
                            
                            echo.
                            echo Deployment completed successfully!
                        """
                    }
                }
            }
        }
    }
    
    post {
        failure {
            script {
                echo '=========================================='
                echo '✗ Pipeline FAILED!'
                echo '=========================================='
                echo "Failed Stage: ${env.STAGE_NAME}"
                echo "Build URL: ${env.BUILD_URL}"
                echo "Contact: ${env.APPROVER_EMAIL}"
                echo '=========================================='
            }
        }
        success {
            script {
                echo '=========================================='
                echo '✓ Pipeline completed SUCCESSFULLY!'
                echo '=========================================='
                echo "Deployment Mode: ${params.DEPLOYMENT_MODE}"
                
                if (params.DEPLOYMENT_MODE == 'Validate Only') {
                    echo '✓ Validation completed - No deployment performed'
                } else if (params.DEPLOYMENT_MODE == 'Validate and Deploy') {
                    echo '✓ Validation and deployment to production completed successfully!'
                }
                echo '=========================================='
            }
        }
        aborted {
            script {
                echo '=========================================='
                echo '⚠ Pipeline was ABORTED!'
                echo '=========================================='
                echo 'Deployment was cancelled or timed out'
                echo "Contact: ${env.APPROVER_EMAIL} for more information"
                echo '=========================================='
            }
        }
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
    }
}