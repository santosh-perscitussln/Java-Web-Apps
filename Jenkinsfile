pipeline {
    agent any

    environment {
        GIT_REPO       = 'https://github.com/santosh-perscitussln/Java-Web-Apps.git'
        BRANCH         = 'main'
        GIT_CRED_ID    = 'github-pat'               // Add PAT credentials in Jenkins
        PROD_HOST      = '3.85.162.30'
        PROD_USER      = 'prod-deploy'
        PROD_CRED_ID   = 'ec2-prod-deploy-ssh'
        TOMCAT_WEBAPPS = '/prod/tomcat/apache-tomcat-9.0.99/webapps'
        TOMCAT_BIN     = '/prod/tomcat/apache-tomcat-9.0.99/bin'
        APP_NAME       = 'Java-Web-Apps'
        APP_PORT       = '8085'
        BACKUP_PATH    = '/prod/backup'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${BRANCH}", 
                    url: "${GIT_REPO}", 
                    credentialsId: "${GIT_CRED_ID}"
            }
        }

        stage('Get Version from Git Tag') {
            steps {
                script {
                    try {
                        VERSION = sh(script: "git describe --tags --abbrev=0", returnStdout: true).trim()
                        echo "Deploying version from tag: ${VERSION}"
                    } catch (Exception e) {
                        echo "⚠️ No Git tags found. Using default version 0.0.1"
                        VERSION = "0.0.1"
                    }
                    env.VERSION = VERSION
                }
            }
        }

        stage('Build WAR') {
            steps {
                sh '''
                    mvn clean package -DskipTests
                    WAR_FILE=target/${APP_NAME}-${VERSION}.war
                    cp target/*.war \${WAR_FILE}
                    echo "WAR file created: \${WAR_FILE}"
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: [env.PROD_CRED_ID]) {
                    sh '''
                        set -e
                        WAR_FILE=target/${APP_NAME}-${VERSION}.war

                        mkdir -p ~/.ssh
                        chmod 700 ~/.ssh
                        ssh-keyscan -H "${PROD_HOST}" >> ~/.ssh/known_hosts

                        echo "Stopping Tomcat..."
                        #ssh ${PROD_USER}@${PROD_HOST} '${TOMCAT_BIN}/shutdown.sh'
                        #ssh ${PROD_USER}@${PROD_HOST} "${TOMCAT_BIN}/shutdown.sh"
                        ssh ${PROD_USER}@${PROD_HOST} "TOMCAT_BIN=${TOMCAT_BIN}; \
                        if pgrep -f 'tomcat' > /dev/null; then \
                        echo 'Tomcat is running. Stopping...'; \
                        \$TOMCAT_BIN/shutdown.sh; \
                        echo 'Tomcat stopped.'; \
                        else \
                        echo 'Tomcat is not running. Skipping shutdown.'; \
                        fi"


                        echo "Backing up current WAR..."
                        ssh ${PROD_USER}@${PROD_HOST} "mkdir -p ${BACKUP_PATH}/$(date +%Y%m%d)"
                        ssh -o StrictHostKeyChecking=no ${PROD_USER}@${PROD_HOST} '
                        if [ -f ${TOMCAT_WEBAPPS}/${APP_NAME}.war ]; then
                            mv ${TOMCAT_WEBAPPS}/${APP_NAME}.war ${BACKUP_PATH}/$(date +%Y%m%d)/${APP_NAME}_backup_$(date +%Y%m%d%H%M%S).war
                        fi
                        '

                        echo "Copying new WAR..."
                        scp "\${WAR_FILE}" ${PROD_USER}@${PROD_HOST}:${TOMCAT_WEBAPPS}/

                        echo "Renaming WAR to standard name..."
                        #ssh ${PROD_USER}@${PROD_HOST} "mv ${TOMCAT_WEBAPPS}/${APP_NAME}-${VERSION}.war ${TOMCAT_WEBAPPS}/${APP_NAME}.war"

                        echo "Starting Tomcat..."
                        ssh ${PROD_USER}@${PROD_HOST} "${TOMCAT_BIN}/startup.sh"
                        sleep 10
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    sh '''
                        for i in $(seq 1 20); do
                          if curl -fsS "http://${PROD_HOST}:${APP_PORT}/${APP_NAME}/" >/dev/null; then
                            echo "✅ App is running!"
                            exit 0
                          fi
                          echo "Waiting for app..."
                          sleep 5
                        done
                        echo "❌ App did not start properly."
                        exit 1
                    '''
                }
            }
        }
    }
}
