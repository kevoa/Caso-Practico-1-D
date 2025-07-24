pipeline {
    // Definimos un agente 'none' a nivel global porque cada etapa definirá el suyo.
    agent none

    environment {
        AWS_REGION = 'us-east-1'
    }

    stages {
        // --- ETAPAS COMUNES A AMBAS RAMAS ---

        stage('Análisis Estático') {
            // Esta etapa se ejecuta SIEMPRE, pero solo en la rama 'develop'
            when { branch 'develop' }
            agent { label 'static-agent' }
            steps {
                sh 'echo "--- Ejecutando en Host: $(hostname) como $(whoami) ---"'
                checkout scm
                
                echo "--- Descargando configuración de Staging ---"
                dir('config') {
                    git url: 'git@github.com:kevoa/todo-list-aws-config.git', branch: 'staging', credentialsId: 'github-ssh-key'
                }

                sh '''
                    echo "--- Instalando y ejecutando análisis estático ---"
                    pip install -q flake8 bandit --break-system-packages
                    /home/jenkins/.local/bin/flake8 ./src --output-file=flake8-report.txt || true
                    /home/jenkins/.local/bin/bandit -r ./src -f html -o bandit-report.html || true
                '''
                
                echo "--- Publicando informes ---"
                publishHTML(target: [ reportDir: '.', reportFiles: 'flake8-report.txt', reportName: 'Informe Flake8' ])
                publishHTML(target: [ reportDir: '.', reportFiles: 'bandit-report.html', reportName: 'Informe Bandit' ])
            }
            post {
                always {
                    stash name: 'config-staging', includes: 'config/**'
                }
            }
        }

        // --- ETAPAS ESPECÍFICAS PARA STAGING (rama develop) ---

        stage('Despliegue en Staging') {
            when { branch 'develop' }
            agent { label 'deploy-test-agent' }
            steps {
                sh 'echo "--- Ejecutando en Host: $(hostname) como $(whoami) ---"'
                checkout scm
                unstash 'config-staging'

                echo "--- Construyendo y desplegando en Staging ---"
                sh 'sam build'
                sh 'sam deploy --config-file config/samconfig.toml --config-env staging || true'

                script {
                    echo "--- Obteniendo y guardando la URL de la API ---"
                    def stackName = 'todo-api-staging'
                    def apiUrl = sh(script: "aws cloudformation describe-stacks --stack-name ${stackName} --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --output text --region ${AWS_REGION}", returnStdout: true).trim()
                    writeFile file: 'api_url.txt', text: apiUrl
                    stash name: 'apiInfo-staging', includes: 'api_url.txt'
                }
            }
        }

        stage('Pruebas de Integración (Staging)') {
            when { branch 'develop' }
            agent { label 'deploy-test-agent' }
            steps {
                sh 'echo "--- Ejecutando en Host: $(hostname) como $(whoami) ---"'
                checkout scm
                unstash 'apiInfo-staging'
                
                script {
                    def apiUrl = readFile('api_url.txt').trim()
                    echo "--- Ejecutando pruebas completas contra: ${apiUrl} ---"
                    sh 'pip install -q -r test/integration/requirements.txt --break-system-packages'
                    sh """
                        export PATH="/home/jenkins/.local/bin:\$PATH"
                        API_URL=${apiUrl} pytest test/integration/todoApiTest.py
                    """
                }
            }
        }

        stage('Promoción a Master') {
            when { branch 'develop' }
            agent { label 'deploy-test-agent' }
            steps {
                sh 'echo "--- Ejecutando en Host: $(hostname) como $(whoami) ---"'
                checkout scm
                
                echo "--- Fusionando 'develop' con 'master' ---"
                sshagent(credentials: ['github-ssh-key']) {
                    sh '''
                        git config --global user.email "jenkins@kevin.com"
                        git config --global user.name "Jenkins CI"
                        git checkout master
                        git merge origin/develop
                        git push origin master
                    '''
                }
            }
        }

        // --- ETAPAS ESPECÍFICAS PARA PRODUCCIÓN (rama master) ---

        stage('Despliegue en Producción') {
            when { branch 'master' }
            agent { label 'deploy-test-agent' }
            steps {
                sh 'echo "--- Ejecutando en Host: $(hostname) como $(whoami) ---"'
                checkout scm
                
                echo "--- Descargando configuración de Producción ---"
                dir('config') {
                    git url: 'git@github.com:kevoa/todo-list-aws-config.git', branch: 'production', credentialsId: 'github-ssh-key'
                }

                echo "--- Construyendo y desplegando en Producción ---"
                sh 'sam build'
                sh 'sam deploy --config-file config/samconfig.toml --config-env production || true'
            }
        }

        stage('Pruebas de Lectura (Producción)') {
            when { branch 'master' }
            agent { label 'deploy-test-agent' }
            steps {
                sh 'echo "--- Ejecutando en Host: $(hostname) como $(whoami) ---"'
                script {
                    echo "--- Instalando dependencias de las pruebas ---"
                    sh 'pip install -q -r test/integration/requirements.txt --break-system-packages'

                    echo "--- Obteniendo la URL del API de Producción ---"
                    def stackName = 'todo-api-production'
                    def apiUrl = sh(script: "aws cloudformation describe-stacks --stack-name ${stackName} --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --output text --region ${AWS_REGION}", returnStdout: true).trim()
                    
                    echo "--- Ejecutando pruebas de solo lectura contra: ${apiUrl} ---"
                    sh """
                        export PATH="/home/jenkins/.local/bin:\$PATH"
                        API_URL=${apiUrl} pytest -m readonly test/integration/todoApiTest.py
                    """
                }
            }
        }
    }
}
