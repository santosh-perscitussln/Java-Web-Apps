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
          environment {
            // make sure these are set (adjust to your paths/host)
            APP_NAME       = 'Java-Web-Apps'
            VERSION        = '0.0.1'                         // from your pipeline/tag if you want
            PROD_USER      = 'prod-deploy'
            PROD_HOST      = '3.85.162.30'
            PROD_CRED_ID   = 'prod-deploy'
            TOMCAT_BIN     = '/prod/tomcat/apache-tomcat-9.0.99/bin'
            TOMCAT_WEBAPPS = '/prod/tomcat/apache-tomcat-9.0.99/webapps'
            BACKUP_DIR     = '/prod/backup'
          }
          steps {
            sshagent(credentials: [env.PROD_CRED_ID]) {
              sh '''#!/bin/bash
        set -euo pipefail
        
        WAR_FILE="target/${APP_NAME}-${VERSION}.war"
        
        # Ensure host key
        mkdir -p ~/.ssh && chmod 700 ~/.ssh
        ssh-keyscan -H "$PROD_HOST" >> ~/.ssh/known_hosts
        
        echo "Stopping Tomcat if running..."
        ssh "${PROD_USER}@${PROD_HOST}" "TOMCAT_BIN='${TOMCAT_BIN}'; if pgrep -f tomcat >/dev/null 2>&1; then '\$TOMCAT_BIN/shutdown.sh' || true; fi"
        
        echo "Preparing backup dir on remote..."
        ssh "${PROD_USER}@${PROD_HOST}" "mkdir -p '${BACKUP_DIR}'"
        
        echo "Copying new WAR to remote as staging file (.war.new)..."
        scp "${WAR_FILE}" "${PROD_USER}@${PROD_HOST}":"${TOMCAT_WEBAPPS}/${APP_NAME}.war.new"
        
        echo "Backup old WAR and swap in new one..."
        ssh "${PROD_USER}@${PROD_HOST}" "
          set -euo pipefail
          APP='${APP_NAME}'
          WEBAPPS='${TOMCAT_WEBAPPS}'
          BACKUPS='${BACKUP_DIR}'
          NOW=\$(date +%Y%m%d%H%M%S)
        
          # backup only existing deployed WAR (APP.war), not the newly uploaded .war.new
          if [ -f \"\$WEBAPPS/\$APP.war\" ]; then
            echo \"Backing up \$WEBAPPS/\$APP.war to \$BACKUPS/\$APP-\$NOW.war\"
            mv \"\$WEBAPPS/\$APP.war\" \"\$BACKUPS/\$APP-\$NOW.war\"
          else
            echo \"No existing \$APP.war to backup. Skipping...\"
          fi
        
          # promote the new file
          mv \"\$WEBAPPS/\$APP.war.new\" \"\$WEBAPPS/\$APP.war\"
        "
        
        echo "Starting Tomcat..."
        ssh "${PROD_USER}@${PROD_HOST}" "'${TOMCAT_BIN}/startup.sh'"
        
        echo "Done."
        '''
            }
          }
        }




                
                echo "Copying new WAR..."
                scp "\${WAR_FILE}" ${PROD_USER}@${env.PROD_HOST}:"\${TOMCAT_WEBAPPS}/"
                
                echo "Renaming WAR to standard name..."
                ssh ${PROD_USER}@$PROD_HOST "mv ${TOMCAT_WEBAPPS}/${APP_NAME}-${VERSION}.war ${TOMCAT_WEBAPPS}/${APP_NAME}.war"
                
                echo "Starting Tomcat..."
                ssh ${PROD_USER}@$$PROD_HOST "${TOMCAT_BIN}/startup.sh"
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
