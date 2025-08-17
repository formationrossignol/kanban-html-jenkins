pipeline {
    agent any
    
    environment {
        APP_NAME = 'kanban-pro'
        BUILD_TIMESTAMP = sh(returnStdout: true, script: 'date +%Y%m%d_%H%M%S || powershell -Command "Get-Date -Format \'yyyyMMdd_HHmmss\'"').trim()
        AVAILABLE_PORT = ''
        APP_PATH = ''
        BACKUP_PATH = ''
        LOG_PATH = ''
        CONFIG_PATH = ''
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    
    stages {
        stage('Détection système et ports') {
            steps {
                echo "Détection du système et des ports disponibles..."
                
                script {
                    // Détecter l'OS
                    def isWindows = false
                    def isMacOS = false
                    def isLinux = false
                    
                    try {
                        def windowsTest = powershell(returnStatus: true, script: 'echo "test"')
                        if (windowsTest == 0) {
                            isWindows = true
                            env.OS_TYPE = 'Windows'
                            env.APP_PATH = 'C:\\inetpub\\wwwroot\\kanban'
                            env.BACKUP_PATH = 'C:\\backups\\kanban'
                            env.LOG_PATH = 'C:\\logs\\kanban'
                            env.CONFIG_PATH = 'C:\\nginx\\conf'
                        }
                    } catch (Exception e) {
                        // Pas Windows
                    }
                    
                    if (!isWindows) {
                        def osName = sh(returnStdout: true, script: 'uname -s').trim()
                        echo "OS détecté par uname: ${osName}"
                        
                        if (osName == 'Darwin') {
                            isMacOS = true
                            env.OS_TYPE = 'macOS'
                            env.APP_PATH = '/Users/' + sh(returnStdout: true, script: 'whoami').trim() + '/kanban'
                            env.BACKUP_PATH = '/Users/' + sh(returnStdout: true, script: 'whoami').trim() + '/kanban-backup'
                            env.LOG_PATH = '/Users/' + sh(returnStdout: true, script: 'whoami').trim() + '/kanban-logs'
                            env.CONFIG_PATH = '/opt/homebrew/etc/nginx'
                            echo "Configuration macOS définie:"
                            echo "APP_PATH: ${env.APP_PATH}"
                        } else {
                            isLinux = true
                            env.OS_TYPE = 'Linux'
                            env.APP_PATH = '/var/www/kanban'
                            env.BACKUP_PATH = '/var/backups/kanban'
                            env.LOG_PATH = '/var/log/nginx'
                            env.CONFIG_PATH = '/etc/nginx'
                            
                            // Détecter la distribution Linux
                            if (fileExists('/etc/os-release')) {
                                env.DISTRO = sh(returnStdout: true, script: '. /etc/os-release && echo $ID').trim()
                            }
                        }
                    }
                    
                    echo "Système détecté: ${env.OS_TYPE}"
                    
                    // Trouver un port disponible
                    def portsToTry = [80, 3000, 4000, 5000, 8081, 8082, 8083, 8084, 8085, 9000, 9001, 9080]
                    def availablePort = null
                    
                    for (port in portsToTry) {
                        def isPortFree = false
                        try {
                            if (env.OS_TYPE == 'Windows') {
                                def result = powershell(returnStatus: true, script: "Test-NetConnection -ComputerName localhost -Port ${port} -InformationLevel Quiet -WarningAction SilentlyContinue")
                                isPortFree = (result != 0) // Port libre si la connexion échoue
                            } else {
                                def result = sh(returnStatus: true, script: "nc -z localhost ${port} 2>/dev/null || netstat -ln | grep ':${port}' >/dev/null 2>&1")
                                isPortFree = (result != 0)
                            }
                        } catch (Exception e) {
                            isPortFree = true // Considérer comme libre en cas d'erreur
                        }
                        
                        if (isPortFree) {
                            availablePort = port
                            break
                        } else {
                            echo "Port ${port} occupé"
                        }
                    }
                    
                    if (availablePort) {
                        env.AVAILABLE_PORT = availablePort.toString()
                        echo "Port disponible trouvé: ${env.AVAILABLE_PORT}"
                    } else {
                        error("Aucun port disponible trouvé")
                    }
                }
            }
        }
        
        stage('Installation des dépendances') {
            steps {
                echo "Installation des dépendances..."
                
                script {
                    switch(env.OS_TYPE) {
                        case 'Windows':
                            powershell '''
                                if (!(Get-Command choco -ErrorAction SilentlyContinue)) {
                                    Set-ExecutionPolicy Bypass -Scope Process -Force
                                    iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
                                }
                                choco install nginx -y
                            '''
                            break
                            
                        case 'macOS':
                            sh '''
                                if ! command -v brew &> /dev/null; then
                                    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
                                fi
                                brew install nginx
                            '''
                            break
                            
                        case 'Linux':
                            switch(env.DISTRO) {
                                case ['ubuntu', 'debian']:
                                    sh 'sudo apt update && sudo apt install -y nginx'
                                    break
                                case ['centos', 'rhel', 'rocky', 'almalinux']:
                                    sh 'sudo yum install -y epel-release && sudo yum install -y nginx'
                                    break
                                case 'fedora':
                                    sh 'sudo dnf install -y nginx'
                                    break
                                default:
                                    sh 'sudo apt update && sudo apt install -y nginx'
                            }
                            break
                    }
                }
            }
        }
        
        stage('Préparation des répertoires') {
            steps {
                echo "Création des répertoires..."
                
                script {
                    switch(env.OS_TYPE) {
                        case 'Windows':
                            powershell '''
                                New-Item -ItemType Directory -Force -Path $env:APP_PATH
                                New-Item -ItemType Directory -Force -Path $env:BACKUP_PATH
                                New-Item -ItemType Directory -Force -Path $env:LOG_PATH
                            '''
                            break
                            
                        case 'macOS':
                            sh '''
                                echo "Variables d'environnement:"
                                echo "APP_PATH: ${APP_PATH}"
                                echo "BACKUP_PATH: ${BACKUP_PATH}"
                                echo "LOG_PATH: ${LOG_PATH}"
                                
                                if [ -z "${APP_PATH}" ]; then
                                    echo "Erreur: APP_PATH est vide, définition manuelle"
                                    USER_HOME=$(echo ~)
                                    APP_PATH="${USER_HOME}/kanban"
                                    BACKUP_PATH="${USER_HOME}/kanban-backup"
                                    LOG_PATH="${USER_HOME}/kanban-logs"
                                fi
                                
                                mkdir -p "${APP_PATH}"
                                mkdir -p "${BACKUP_PATH}"
                                mkdir -p "${LOG_PATH}"
                                
                                echo "Répertoires créés avec succès"
                                echo "APP_PATH final: ${APP_PATH}"
                                ls -la "${APP_PATH}"
                            '''
                            break
                            
                        case 'Linux':
                            sh '''
                                sudo mkdir -p ${APP_PATH} ${BACKUP_PATH} ${LOG_PATH}
                                sudo chown -R www-data:www-data ${APP_PATH}
                                sudo chown -R jenkins:jenkins ${BACKUP_PATH}
                            '''
                            break
                    }
                }
            }
        }
        
        stage('Récupération du code') {
            steps {
                echo "Récupération du code source..."
                cleanWs()
                checkout scm
                
                script {
                    if (!fileExists('kanban.html')) {
                        error("Fichier kanban.html manquant")
                    }
                    echo "Fichier kanban.html trouvé"
                }
            }
        }
        
        stage('Préparation des fichiers') {
            steps {
                echo "Préparation des fichiers..."
                
                script {
                    if (env.OS_TYPE == 'Windows') {
                        powershell '''
                            New-Item -ItemType Directory -Force -Path "build"
                            Copy-Item "kanban.html" "build/index.html"
                            
                            @"
Build: $env:BUILD_NUMBER
Date: $env:BUILD_TIMESTAMP
Port: $env:AVAILABLE_PORT
OS: Windows
"@ | Out-File -FilePath "build/version.txt" -Encoding UTF8
                        '''
                    } else {
                        sh '''
                            mkdir -p build
                            cp kanban.html build/index.html
                            
                            cat > build/version.txt << EOF
Build: ${BUILD_NUMBER}
Date: ${BUILD_TIMESTAMP}
Port: ${AVAILABLE_PORT}
OS: ${OS_TYPE}
EOF
                        '''
                    }
                }
            }
        }
        
        stage('Configuration Nginx') {
            steps {
                echo "Configuration de Nginx sur le port ${AVAILABLE_PORT}..."
                
                script {
                    switch(env.OS_TYPE) {
                        case 'Windows':
                            powershell '''
                                $nginxConfig = @"
events {
    worker_connections 1024;
}

http {
    include mime.types;
    default_type application/octet-stream;
    
    server {
        listen $env:AVAILABLE_PORT;
        server_name localhost;
        root $($env:APP_PATH.Replace('\\', '/'));
        index index.html;
        
        location / {
            try_files `$uri `$uri/ /index.html;
        }
    }
}
"@
                                $nginxConfig | Out-File -FilePath "$env:CONFIG_PATH\\nginx.conf" -Encoding UTF8
                            '''
                            break
                            
                        case 'macOS':
                            sh '''
                                # Détecter le chemin Nginx (Homebrew Intel vs Apple Silicon)
                                if [ -f "/opt/homebrew/etc/nginx/nginx.conf" ]; then
                                    CONFIG_PATH="/opt/homebrew/etc/nginx"
                                elif [ -f "/usr/local/etc/nginx/nginx.conf" ]; then
                                    CONFIG_PATH="/usr/local/etc/nginx"
                                else
                                    echo "Configuration Nginx non trouvée, installation requise"
                                    exit 1
                                fi
                                
                                USER_HOME=$(echo ~)
                                APP_PATH="${USER_HOME}/kanban"
                                
                                cat > ${CONFIG_PATH}/nginx.conf << EOF
events {
    worker_connections 1024;
}

http {
    include mime.types;
    default_type application/octet-stream;
    
    server {
        listen ${AVAILABLE_PORT};
        server_name localhost;
        root ${APP_PATH};
        index index.html;
        
        location / {
            try_files \\$uri \\$uri/ /index.html;
        }
    }
}
EOF
                                echo "Configuration Nginx créée dans: ${CONFIG_PATH}/nginx.conf"
                            '''
                            break
                            
                        case 'Linux':
                            sh '''
                                sudo tee ${CONFIG_PATH}/nginx.conf > /dev/null << EOF
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    server {
        listen ${AVAILABLE_PORT};
        server_name localhost;
        root ${APP_PATH};
        index index.html;
        
        location / {
            try_files \\$uri \\$uri/ /index.html;
        }
    }
}
EOF
                                sudo rm -f /etc/nginx/sites-enabled/default
                            '''
                            break
                    }
                }
            }
        }
        
        stage('Sauvegarde') {
            steps {
                echo "Sauvegarde de l'existant..."
                
                script {
                    switch(env.OS_TYPE) {
                        case 'Windows':
                            powershell '''
                                if (Test-Path $env:APP_PATH\\index.html) {
                                    $backupName = "backup_$env:BUILD_TIMESTAMP"
                                    Copy-Item -Path $env:APP_PATH -Destination "$env:BACKUP_PATH\\$backupName" -Recurse -Force
                                }
                            '''
                            break
                            
                        default:
                            sh '''
                                if [ -f ${APP_PATH}/index.html ]; then
                                    BACKUP_NAME="backup_${BUILD_TIMESTAMP}"
                                    if [ "${OS_TYPE}" = "Linux" ]; then
                                        sudo cp -r ${APP_PATH} ${BACKUP_PATH}/${BACKUP_NAME}
                                    else
                                        cp -r ${APP_PATH} ${BACKUP_PATH}/${BACKUP_NAME}
                                    fi
                                fi
                            '''
                            break
                    }
                }
            }
        }
        
        stage('Déploiement') {
            steps {
                echo "Déploiement de l'application..."
                
                script {
                    switch(env.OS_TYPE) {
                        case 'Windows':
                            powershell '''
                                Stop-Process -Name "nginx" -Force -ErrorAction SilentlyContinue
                                Copy-Item -Path "build\\*" -Destination $env:APP_PATH -Recurse -Force
                                Start-Process -FilePath "nginx" -WorkingDirectory "C:\\tools\\nginx" -WindowStyle Hidden
                            '''
                            break
                            
                        case 'macOS':
                            sh '''
                                brew services stop nginx 2>/dev/null || true
                                
                                USER_HOME=$(echo ~)
                                APP_PATH="${USER_HOME}/kanban"
                                
                                cp -r build/* "${APP_PATH}/"
                                
                                brew services start nginx
                                echo "Application déployée dans: ${APP_PATH}"
                            '''
                            break
                            
                        case 'Linux':
                            sh '''
                                sudo systemctl stop nginx 2>/dev/null || true
                                sudo cp -r build/* ${APP_PATH}/
                                sudo chown -R www-data:www-data ${APP_PATH}
                                sudo systemctl start nginx
                                sudo systemctl enable nginx
                            '''
                            break
                    }
                }
            }
        }
        
        stage('Vérification') {
            steps {
                echo "Vérification du déploiement..."
                
                script {
                    def testUrl = "http://localhost:${env.AVAILABLE_PORT}"
                    
                    switch(env.OS_TYPE) {
                        case 'Windows':
                            powershell """
                                Start-Sleep -Seconds 3
                                try {
                                    \$response = Invoke-WebRequest -Uri "${testUrl}" -UseBasicParsing -TimeoutSec 10
                                    if (\$response.StatusCode -eq 200) {
                                        Write-Host "Application accessible sur ${testUrl}"
                                    } else {
                                        throw "HTTP \$(\$response.StatusCode)"
                                    }
                                } catch {
                                    throw "Vérification échouée: \$_"
                                }
                            """
                            break
                            
                        default:
                            sh """
                                sleep 3
                                HTTP_CODE=\$(curl -s -o /dev/null -w "%{http_code}" ${testUrl} || echo "000")
                                if [ "\$HTTP_CODE" = "200" ]; then
                                    echo "Application accessible sur ${testUrl}"
                                else
                                    echo "Erreur HTTP: \$HTTP_CODE"
                                    exit 1
                                fi
                            """
                            break
                    }
                }
            }
        }
        
        stage('Scripts de maintenance') {
            steps {
                echo "Création des scripts de maintenance..."
                
                script {
                    switch(env.OS_TYPE) {
                        case 'Windows':
                            powershell '''
                                $controlScript = @"
param([string]`$Action)

switch (`$Action) {
    "start" { Start-Process nginx -WorkingDirectory "C:\\tools\\nginx" }
    "stop" { Stop-Process -Name nginx -Force }
    "restart" { 
        Stop-Process -Name nginx -Force -ErrorAction SilentlyContinue
        Start-Process nginx -WorkingDirectory "C:\\tools\\nginx"
    }
    "status" { Get-Process nginx -ErrorAction SilentlyContinue }
    default { Write-Host "Usage: script.ps1 {start|stop|restart|status}" }
}
"@
                                $controlScript | Out-File -FilePath "C:\\scripts\\kanban-control.ps1" -Encoding UTF8
                                New-Item -ItemType Directory -Force -Path "C:\\scripts"
                            '''
                            break
                            
                        default:
                            sh '''
                                if [ "${OS_TYPE}" = "Linux" ]; then
                                    sudo tee /usr/local/bin/kanban-control.sh > /dev/null << 'EOF'
#!/bin/bash
case "$1" in
    start) sudo systemctl start nginx ;;
    stop) sudo systemctl stop nginx ;;
    restart) sudo systemctl restart nginx ;;
    status) sudo systemctl status nginx --no-pager ;;
    *) echo "Usage: $0 {start|stop|restart|status}" ;;
esac
EOF
                                    sudo chmod +x /usr/local/bin/kanban-control.sh
                                else
                                    cat > /usr/local/bin/kanban-control.sh << 'EOF'
#!/bin/bash
case "$1" in
    start) brew services start nginx ;;
    stop) brew services stop nginx ;;
    restart) brew services restart nginx ;;
    status) brew services list | grep nginx ;;
    *) echo "Usage: $0 {start|stop|restart|status}" ;;
esac
EOF
                                    chmod +x /usr/local/bin/kanban-control.sh
                                fi
                            '''
                            break
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        
        success {
            echo "Déploiement réussi!"
            echo "Application disponible sur: http://localhost:${AVAILABLE_PORT}"
            echo "Système: ${OS_TYPE}"
            echo "Répertoire: ${APP_PATH}"
        }
        
        failure {
            echo "Déploiement échoué!"
            
            script {
                try {
                    switch(env.OS_TYPE) {
                        case 'Windows':
                            powershell '''
                                $latestBackup = Get-ChildItem $env:BACKUP_PATH -Directory | Sort-Object CreationTime -Descending | Select-Object -First 1
                                if ($latestBackup) {
                                    Copy-Item "$($latestBackup.FullName)\\*" $env:APP_PATH -Recurse -Force
                                    Write-Host "Restauration effectuée"
                                }
                            '''
                            break
                            
                        default:
                            sh '''
                                LATEST_BACKUP=$(ls -t ${BACKUP_PATH}/ 2>/dev/null | head -1)
                                if [ ! -z "$LATEST_BACKUP" ]; then
                                    if [ "${OS_TYPE}" = "Linux" ]; then
                                        sudo cp -r ${BACKUP_PATH}/$LATEST_BACKUP/* ${APP_PATH}/
                                    else
                                        cp -r ${BACKUP_PATH}/$LATEST_BACKUP/* ${APP_PATH}/
                                    fi
                                    echo "Restauration effectuée"
                                fi
                            '''
                            break
                    }
                } catch (Exception e) {
                    echo "Restauration échouée: ${e.getMessage()}"
                }
            }
        }
    }
}
