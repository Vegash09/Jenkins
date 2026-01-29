pipeline {
    agent any
    
    environment {
        DATABRICKS_CLIENT_ID = 'e1720236-6ef3-47c5-b73e-700d4a171681'
        DATABRICKS_CLIENT_SECRET = 'dosed0e6ab7e4b30bb5ba34f5b3971c88456'
        DATABRICKS_HOST = 'https://adb-7405617499680447.7.azuredatabricks.net/'
        DATABRICKS_BUNDLE_ENV = 'production'
        BUNDLE_VAR_DATABRICKS_CLIENT_ID = 'e1720236-6ef3-47c5-b73e-700d4a171681'
        DATABRICKS_CLI_PATH = 'C:\\databricks-cli'
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
                    powershell '''
                        $cliPath = "C:\\databricks-cli"
                        $cliExe = "$cliPath\\databricks.exe"
                        
                        # Create directory if it doesn't exist
                        if (!(Test-Path $cliPath)) {
                            New-Item -ItemType Directory -Path $cliPath -Force | Out-Null
                            Write-Host "Created directory: $cliPath"
                        }
                        
                        # Check if CLI already exists
                        if (Test-Path $cliExe) {
                            Write-Host "Databricks CLI already installed at: $cliExe"
                            & $cliExe version
                        } else {
                            Write-Host "Downloading Databricks CLI..."
                            
                            # Download the latest Windows CLI
                            $downloadUrl = "https://github.com/databricks/cli/releases/latest/download/databricks_cli_windows_amd64.zip"
                            $zipFile = "$env:TEMP\\databricks_cli.zip"
                            
                            try {
                                Invoke-WebRequest -Uri $downloadUrl -OutFile $zipFile -UseBasicParsing
                                Write-Host "Downloaded CLI to: $zipFile"
                                
                                # Extract
                                Write-Host "Extracting CLI..."
                                Expand-Archive -Path $zipFile -DestinationPath $cliPath -Force
                                
                                # Verify
                                if (Test-Path $cliExe) {
                                    Write-Host "Successfully installed Databricks CLI!"
                                    & $cliExe version
                                } else {
                                    throw "CLI executable not found after extraction"
                                }
                                
                                # Cleanup
                                Remove-Item $zipFile -Force
                            } catch {
                                Write-Error "Failed to install Databricks CLI: $_"
                                exit 1
                            }
                        }
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
                        
                        Get-ChildItem -Path $env:WORKSPACE -Filter "*.py" -Recurse | ForEach-Object {
                            $file = $_.FullName
                            Write-Host "Found: $($_.Name)"
                            
                            try {
                                $content = Get-Content $file -Raw -ErrorAction Stop
                                
                                if ($content -notmatch "^# Databricks notebook source") {
                                    $newContent = "$header`n$content"
                                    Set-Content -Path $file -Value $newContent -NoNewline
                                    Write-Host "  Added header to: $($_.Name)" -ForegroundColor Green
                                    $processedCount++
                                } else {
                                    Write-Host "  Header already exists, skipped" -ForegroundColor Yellow
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
                    bat '''
                        set PATH=%PATH%;C:\\databricks-cli
                        
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
                        set PATH=%PATH%;C:\\databricks-cli
                        
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