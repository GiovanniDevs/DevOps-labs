#!/user/bin/env groovy

// library name from jenkins system settings
@Library('Jenkins-SL')

def gv

pipeline {
    agent any
    tools {
        maven 'maven-3.9'   // must match Manage Jenkins → Tools exactly
    }

    stages {
        stage('init') {
            steps {
                script {
                    gv = load 'script.groovy'
                }
            }
        }
        stage('build jar') {
            steps {
                script {
                    buildJar()
                }
            }
        }
        stage('build and push image') {
            steps {
                script {
                    buildImage 'giovannidevs/demo-app:jma-2.0'
                    dockerLogin()
                    dockerPush 'giovannidevs/demo-app:jma-2.0'
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    gv.deployApp()
                }
            }
        }
    }
}
