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
                        setlocal enabledelayedexpansion
                        
                        set "CLI_PATH=%DATABRICKS_CLI_PATH%"
                        set "CLI_EXE=%CLI_PATH%\\databricks.exe"
                        
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
                        set "VERSION=v0.234.0"
                        set "VERSION_NUM=0.234.0"
                        set "DOWNLOAD_URL=https://github.com/databricks/cli/releases/download/%VERSION%/databricks_cli_%VERSION_NUM%_windows_amd64.zip"
                        set "ZIP_FILE=%TEMP%\\databricks_cli.zip"
                        
                        echo Download URL: %DOWNLOAD_URL%
                        
                        REM Download using curl (available in Windows 10+)
                        curl -L "%DOWNLOAD_URL%" -o "%ZIP_FILE%"
                        if errorlevel 1 (
                            echo ERROR: Download failed!
                            exit /b 1
                        )
                        echo Downloaded successfully!
                        
                        REM Extract using PowerShell
                        echo Extracting...
                        powershell -Command "Expand-Archive -Path '%ZIP_FILE%' -DestinationPath '%CLI_PATH%' -Force"
                        if errorlevel 1 (
                            echo ERROR: Extraction failed!
                            exit /b 1
                        )
                        
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
                        if exist "%ZIP_FILE%" del "%ZIP_FILE%"
                        
                        echo Setup complete!
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
                            
                            REM Verify credentials are set
                            echo DATABRICKS_HOST: %DATABRICKS_HOST%
                            
                            REM Show only first 8 chars of client ID
                            set "CLIENT_ID=%DATABRICKS_CLIENT_ID%"
                            set "CLIENT_ID_PREVIEW=%CLIENT_ID:~0,8%"
                            echo DATABRICKS_CLIENT_ID: %CLIENT_ID_PREVIEW%...
                            
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
                    bat '''
                        @echo off
                        setlocal enabledelayedexpansion
                        
                        set "HEADER=# Databricks notebook source"
                        set /a PROCESSED_COUNT=0
                        set /a SKIPPED_COUNT=0
                        
                        echo Scanning for Python files in: %WORKSPACE%
                        
                        REM Find all .py files
                        for /r "%WORKSPACE%" %%f in (*.py) do (
                            echo Found: %%~nxf
                            
                            REM Check if file already has header
                            findstr /b /c:"# Databricks notebook source" "%%f" >nul
                            if !errorlevel! equ 0 (
                                echo   Already has header
                                set /a SKIPPED_COUNT+=1
                            ) else (
                                REM Add header
                                echo !HEADER!> "%%f.tmp"
                                type "%%f" >> "%%f.tmp"
                                move /y "%%f.tmp" "%%f" >nul
                                echo   Added header
                                set /a PROCESSED_COUNT+=1
                            )
                        )
                        
                        echo.
                        echo Processed: !PROCESSED_COUNT! ^| Skipped: !SKIPPED_COUNT!
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
                            set "PATH=%PATH%;%DATABRICKS_CLI_PATH%"
                            set "DATABRICKS_HOST=%DATABRICKS_HOST%"
                            set "DATABRICKS_CLIENT_ID=%DATABRICKS_CLIENT_ID%"
                            set "DATABRICKS_CLIENT_SECRET=%DATABRICKS_CLIENT_SECRET%"
                            
                            echo.
                            echo ========================================
                            echo Running: databricks bundle validate
                            echo ========================================
                            echo.
                            
                            databricks bundle validate
                            if errorlevel 1 (
                                echo ERROR: Validation failed!
                                exit /b 1
                            )
                            
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
                            set "PATH=%PATH%;%DATABRICKS_CLI_PATH%"
                            set "DATABRICKS_HOST=%DATABRICKS_HOST%"
                            set "DATABRICKS_CLIENT_ID=%DATABRICKS_CLIENT_ID%"
                            set "DATABRICKS_CLIENT_SECRET=%DATABRICKS_CLIENT_SECRET%"
                            set "BUNDLE_VAR_DATABRICKS_CLIENT_ID=%DATABRICKS_CLIENT_ID%"
                            
                            echo.
                            echo ========================================
                            echo Running: databricks bundle deploy
                            echo ========================================
                            echo.
                            
                            databricks bundle deploy
                            if errorlevel 1 (
                                echo ERROR: Deployment failed!
                                exit /b 1
                            )
                            
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