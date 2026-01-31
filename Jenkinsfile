pipeline {
    agent any
    
    environment {
        DATABRICKS_BUNDLE_ENV = 'production'
        DATABRICKS_CLI_PATH = "${WORKSPACE}\\.databricks-cli"
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
                    bat '''
                        @echo off
                        
                        set CLI_PATH=%DATABRICKS_CLI_PATH%
                        set CLI_EXE=%CLI_PATH%\\databricks.exe
                        
                        REM Create directory if it doesn't exist
                        if not exist "%CLI_PATH%" (
                            mkdir "%CLI_PATH%"
                            echo Created directory: %CLI_PATH%
                        )
                        
                        REM Check if already installed
                        if exist "%CLI_EXE%" (
                            echo Databricks CLI already exists!
                            "%CLI_EXE%" version
                            exit /b 0
                        )
                        
                        echo Downloading Databricks CLI...
                        
                        REM Use the correct URL format for Windows
                        set VERSION=v0.234.0
                        set VERSION_NUM=0.234.0
                        set DOWNLOAD_URL=https://github.com/databricks/cli/releases/download/%VERSION%/databricks_cli_%VERSION_NUM%_windows_amd64.zip
                        set ZIP_FILE=%TEMP%\\databricks_cli.zip
                        
                        echo Download URL: %DOWNLOAD_URL%
                        
                        REM Download using PowerShell
                        powershell -Command "Invoke-WebRequest -Uri '%DOWNLOAD_URL%' -OutFile '%ZIP_FILE%'"
                        echo Downloaded successfully!
                        
                        REM Extract using PowerShell
                        echo Extracting...
                        powershell -Command "Expand-Archive -Path '%ZIP_FILE%' -DestinationPath '%CLI_PATH%' -Force"
                        
                        REM Verify extraction
                        if exist "%CLI_EXE%" (
                            echo Installation successful!
                            "%CLI_EXE%" version
                        ) else (
                            echo ERROR: databricks.exe not found after extraction!
                            echo Contents of %CLI_PATH%:
                            dir "%CLI_PATH%"
                            exit /b 1
                        )
                        
                        REM Cleanup
                        del "%ZIP_FILE%"
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
                        bat '''
                            @echo off
                            echo Setting Databricks environment variables...
                            echo DATABRICKS_HOST: %DATABRICKS_HOST%
                            echo DATABRICKS_CLIENT_ID: %DATABRICKS_CLIENT_ID:~0,8%...
                            echo Authentication configured successfully!
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
                        
                        # Find all .py files
                        Get-ChildItem -Path $env:WORKSPACE -Filter "*.py" -Recurse -File | ForEach-Object {
                            Write-Host "Found: $($_.Name)"
                            
                            # Check if file already has header
                            $firstLine = Get-Content $_.FullName -First 1
                            if ($firstLine -match "^# Databricks notebook source") {
                                Write-Host "  Already has header"
                                $skippedCount++
                            } else {
                                # Add header
                                $content = Get-Content $_.FullName -Raw
                                "$header`n$content" | Set-Content $_.FullName
                                Write-Host "  Added header"
                                $processedCount++
                            }
                        }
                        
                        Write-Host ""
                        Write-Host "Processed: $processedCount | Skipped: $skippedCount"
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
                            @echo off
                            set PATH=%PATH%;%DATABRICKS_CLI_PATH%
                            set DATABRICKS_HOST=%DATABRICKS_HOST%
                            set DATABRICKS_CLIENT_ID=%DATABRICKS_CLIENT_ID%
                            set DATABRICKS_CLIENT_SECRET=%DATABRICKS_CLIENT_SECRET%
                            
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
                        bat """
                            @echo off
                            set PATH=%PATH%;%DATABRICKS_CLI_PATH%
                            set DATABRICKS_HOST=%DATABRICKS_HOST%
                            set DATABRICKS_CLIENT_ID=%DATABRICKS_CLIENT_ID%
                            set DATABRICKS_CLIENT_SECRET=%DATABRICKS_CLIENT_SECRET%
                            set BUNDLE_VAR_DATABRICKS_CLIENT_ID=%DATABRICKS_CLIENT_ID%
                            
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
