pipeline {
    agent any
    
    environment {
        // Databricks credentials
        DATABRICKS_CLIENT_ID = 'e1720236-6ef3-47c5-b73e-700d4a171681'
        DATABRICKS_CLIENT_SECRET = 'dosed0e6ab7e4b30bb5ba34f5b3971c88456'
        DATABRICKS_HOST = 'https://adb-7405617499680447.7.azuredatabricks.net/'
        DATABRICKS_BUNDLE_ENV = 'production'
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
        
        stage('Install Databricks CLI') {
            steps {
                script {
                    echo 'Installing Databricks CLI...'
                    bat '''
                        echo Installing Databricks CLI using pip...
                        pip install databricks-cli --upgrade
                        echo.
                        echo Verifying installation...
                        databricks --version
                        echo.
                        echo Databricks CLI installed successfully!
                    '''
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
                        Write-Host ""
                        
                        Get-ChildItem -Path $env:WORKSPACE -Filter "*.py" -Recurse | ForEach-Object {
                            $file = $_.FullName
                            Write-Host "Found: $($_.Name)"
                            
                            try {
                                $content = Get-Content $file -Raw -ErrorAction Stop
                                
                                if ($content -notmatch "^# Databricks notebook source") {
                                    $newContent = "$header`n$content"
                                    Set-Content -Path $file -Value $newContent -NoNewline
                                    Write-Host "  -> Added header to: $($_.Name)" -ForegroundColor Green
                                    $processedCount++
                                } else {
                                    Write-Host "  -> Header already exists, skipped" -ForegroundColor Yellow
                                    $skippedCount++
                                }
                            } catch {
                                Write-Host "  -> Error processing file: $_" -ForegroundColor Red
                            }
                        }
                        
                        Write-Host ""
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host "Summary:" -ForegroundColor Cyan
                        Write-Host "  Files processed: $processedCount" -ForegroundColor Green
                        Write-Host "  Files skipped: $skippedCount" -ForegroundColor Yellow
                        Write-Host "========================================" -ForegroundColor Cyan
                    '''
                }
            }
        }
        
        stage('Validate Databricks Bundle') {
            steps {
                script {
                    echo 'Validating Databricks asset bundle...'
                    bat '''
                        echo.
                        echo ========================================
                        echo Running: databricks bundle validate
                        echo ========================================
                        echo.
                        databricks bundle validate
                        echo.
                        echo ========================================
                        echo Validation completed successfully!
                        echo ========================================
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
                    bat '''
                        echo.
                        echo ========================================
                        echo Running: databricks bundle deploy
                        echo ========================================
                        echo.
                        databricks bundle deploy
                        echo.
                        echo ========================================
                        echo Deployment completed successfully!
                        echo ========================================
                    '''
                }
            }
        }
    }
    
    post {
        failure {
            echo '=========================================='
            echo 'Pipeline FAILED!'
            echo '=========================================='
            echo 'Check the console output above for errors.'
        }
        success {
            echo '=========================================='
            echo 'Pipeline completed SUCCESSFULLY!'
            echo '=========================================='
            echo 'Your Databricks bundle has been deployed.'
        }
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
    }
}
