pipeline {
    agent any
    
    environment {
        DATABRICKS_BUNDLE_ENV = 'production'
        DATABRICKS_CLI_PATH = "${WORKSPACE}\\.databricks-cli"
        DATABRICKS_HOST_URL = 'https://adb-7405617499680447.7.azuredatabricks.net'
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
                        
                        if not exist "%CLI_PATH%" (
                            mkdir "%CLI_PATH%"
                            echo Created directory: %CLI_PATH%
                        )
                        
                        if exist "%CLI_EXE%" (
                            echo Databricks CLI already exists!
                            "%CLI_EXE%" version
                            exit /b 0
                        )
                        
                        echo Downloading Databricks CLI...
                        
                        set "VERSION=v0.234.0"
                        set "VERSION_NUM=0.234.0"
                        set "DOWNLOAD_URL=https://github.com/databricks/cli/releases/download/%VERSION%/databricks_cli_%VERSION_NUM%_windows_amd64.zip"
                        set "ZIP_FILE=%TEMP%\\databricks_cli.zip"
                        
                        echo Download URL: %DOWNLOAD_URL%
                        
                        curl -L "%DOWNLOAD_URL%" -o "%ZIP_FILE%"
                        if errorlevel 1 (
                            echo ERROR: Download failed!
                            exit /b 1
                        )
                        echo Downloaded successfully!
                        
                        echo Extracting...
                        powershell -Command "Expand-Archive -Path '%ZIP_FILE%' -DestinationPath '%CLI_PATH%' -Force"
                        if errorlevel 1 (
                            echo ERROR: Extraction failed!
                            exit /b 1
                        )
                        
                        if exist "%CLI_EXE%" (
                            echo Installation successful!
                            "%CLI_EXE%" version
                        ) else (
                            echo ERROR: databricks.exe not found after extraction!
                            dir "%CLI_PATH%"
                            exit /b 1
                        )
                        
                        if exist "%ZIP_FILE%" del "%ZIP_FILE%"
                        
                        echo Setup complete!
                    '''
                }
            }
        }
        
        stage('Generate databricks.yaml') {
            steps {
                script {
                    echo 'Creating databricks.yaml from configuration files...'
                    bat '''
                        @echo off
                        setlocal enabledelayedexpansion
                        
                        echo Generating databricks.yaml...
                        
                        (
                            echo bundle:
                            echo   name: Deal Share Project
                            echo.
                            echo include:
                            echo   - "config/jobs.yml"
                            echo   - "config/pipelines.yml"
                            echo.
                            echo targets:
                            echo   production:
                            echo     mode: production
                            echo     default: true
                            echo     workspace:
                            echo       host: %DATABRICKS_HOST_URL%
                            echo       root_path: /Workspace/Shared/$${bundle.target}
                            echo.
                            echo     variables:
                            echo       databricks_sp_name: "jenkins-deployment-sp"
                        ) > databricks.yaml
                        
                        echo.
                        echo ========================================
                        echo Generated databricks.yaml:
                        echo ========================================
                        type databricks.yaml
                        echo.
                        echo ========================================
                        
                        echo Verifying config files exist...
                        if exist "config\\jobs.yml" (
                            echo ✓ config\\jobs.yml found
                        ) else (
                            echo ✗ config\\jobs.yml NOT found!
                            exit /b 1
                        )
                        
                        if exist "config\\pipelines.yml" (
                            echo ✓ config\\pipelines.yml found
                        ) else (
                            echo ✗ config\\pipelines.yml NOT found!
                            exit /b 1
                        )
                        
                        echo.
                        echo databricks.yaml generated successfully!
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
                            
                            set "CLIENT_ID=%DATABRICKS_CLIENT_ID%"
                            set "CLIENT_ID_PREVIEW=%CLIENT_ID:~0,8%"
                            echo DATABRICKS_CLIENT_ID: %CLIENT_ID_PREVIEW%...
                            
                            echo Authentication configured successfully!
                        '''
                        
                        env.DATABRICKS_HOST = DATABRICKS_HOST
                        env.DATABRICKS_CLIENT_ID = DATABRICKS_CLIENT_ID
                        env.DATABRICKS_CLIENT_SECRET = DATABRICKS_CLIENT_SECRET
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
                        
                        echo Scanning for Python files in: %WORKSPACE%\\notebooks
                        
                        if not exist "%WORKSPACE%\\notebooks" (
                            echo WARNING: notebooks folder not found!
                            exit /b 0
                        )
                        
                        for /r "%WORKSPACE%\\notebooks" %%f in (*.py) do (
                            echo Found: %%~nxf
                            
                            findstr /b /c:"# Databricks notebook source" "%%f" >nul
                            if !errorlevel! equ 0 (
                                echo   Already has header
                                set /a SKIPPED_COUNT+=1
                            ) else (
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
                            echo Validating Bundle Configuration
                            echo ========================================
                            echo.
                            
                            databricks bundle validate -t production
                            if errorlevel 1 (
                                echo ERROR: Validation failed!
                                exit /b 1
                            )
                            
                            echo.
                            echo ========================================
                            echo Bundle Summary
                            echo ========================================
                            databricks bundle summary -t production
                            
                            echo.
                            echo Validation completed successfully!
                        """
                    }
                }
            }
        }
        
        stage('Show Deployment Plan') {
            steps {
                script {
                    echo 'Showing what will be deployed...'
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
                            echo Deployment Plan
                            echo ========================================
                            echo.
                            echo The following resources will be deployed:
                            echo.
                            
                            databricks bundle summary -t production
                            
                            echo.
                            echo ========================================
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
                    echo 'Deploying Databricks bundle to production...'
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
                            echo Deploying to Production
                            echo ========================================
                            echo.
                            
                            databricks bundle deploy -t production
                            if errorlevel 1 (
                                echo ERROR: Deployment failed!
                                exit /b 1
                            )
                            
                            echo.
                            echo ========================================
                            echo Deployment Completed Successfully!
                            echo ========================================
                            echo.
                            echo Deployed Resources:
                            echo   ✓ Notebooks synced to workspace
                            echo   ✓ Jobs created/updated
                            echo   ✓ Pipelines created/updated
                            echo.
                            echo Location: /Workspace/Shared/production
                            echo.
                        """
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo 'Verifying deployed resources...'
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
                            echo Verifying Deployed Jobs
                            echo ========================================
                            databricks jobs list --output table
                            
                            echo.
                            echo ========================================
                            echo Verifying Deployed Pipelines
                            echo ========================================
                            databricks pipelines list --output table
                            
                            echo.
                            echo Verification complete!
                        """
                    }
                }
            }
        }
    }
    
    post {
        failure {
            echo '=========================================='
            echo '❌ Pipeline FAILED!'
            echo 'Check logs above for errors'
            echo '=========================================='
        }
        success {
            echo '=========================================='
            echo '✅ Pipeline completed SUCCESSFULLY!'
            echo ''
            echo 'Deployed to Production:'
            echo '  ✓ Notebooks'
            echo '  ✓ Jobs'
            echo '  ✓ Pipelines'
            echo ''
            echo 'Workspace: /Workspace/Shared/production'
            echo '=========================================='
        }
        always {
            echo 'Cleaning up workspace...'
            // Keep databricks.yaml for debugging if needed
            bat '''
                @echo off
                if exist databricks.yaml (
                    echo Archiving databricks.yaml...
                    copy databricks.yaml databricks.yaml.last
                )
            '''
            cleanWs()
        }
    }
}