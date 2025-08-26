pipeline {
    agent any

    environment {
        GIT_REPO       = 'https://github.com/santosh-perscitussln/Java-Web-Apps.git'
        BRANCH         = 'main'
        GIT_CRED_ID    = 'github-pat'
        PROD_HOST      = '3.85.162.30'
        PROD_USER      = 'prod-deploy'
        PROD_CRED_ID   = 'ec2-prod-deploy-ssh'
        TOMCAT_WEBAPPS = '/prod/tomcat/apache-tomcat-9.0.99/webapps'
        TOMCAT_BIN     = '/prod/tomcat/apache-tomcat-9.0.99/bin'
        APP_NAME       = 'Java-Web-Apps'
        APP_PORT       = '8085'
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
                    VERSION = sh(script: "git describe --tags --abbrev=0", returnStdout: true).trim()
                    echo "Deploying version: ${VERSION}"
                    env.VERSION = VERSION
                }
            }
        }

        stage('Build WAR') {
            steps {
                sh """
                    mvn clean package -DskipTests
                    WAR_FILE=target_
