pipeline {
    agent any
    environment {
        NEXUS_USER = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus')
        NEXUS_REPO = credentials('nexus-repo-url-prvt')
    }
    stages {
        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Build Artifact') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $NEXUS_REPO/myapp:latest .'
            }
        }
        stage('Log into Nexus Repo') {
            steps {
                sh 'docker login --username $NEXUS_USER --password $NEXUS_PASSWORD $NEXUS_REPO'
            }
        }
        stage('Push to Nexus Repo') {
            steps {
                sh 'docker push $NEXUS_REPO/myapp:latest'
            }
        }
        stage('Deploy to stage') {
            steps {
                sshagent(['ansible-key']) {
                    sh 'ssh -t -t ec2-user@10.0.1.139 -o strictHostKeyChecking=no "cd /etc/ansible && ansible-playbook -i /etc/ansible/stage-hosts stage-env-playbook.yml"'
                }
            }
        }
        //stage('slack notification') {
        //    steps {
        //        slackSend channel: 'jenkins-alert',
        //        message: 'App deployed to Stage, needs approval to deploy to prod',
        //        teamDomain: 'paceu1',
        //        tokenCredentialId: 'slack-credentials'
        //    }
        //}
        stage('Request for Approval') {
            steps {
                timeout(activity: true, time: 10) {
                    input message: 'Needs Approval ', submitter: 'admin'
                }
            }
        }
        stage('Deploy to prod') {
            steps {
                sshagent(['ansible-key']) {
                    sh 'ssh -t -t ec2-user@10.0.1.139 -o strictHostKeyChecking=no "cd /etc/ansible && ansible-playbook -i /etc/ansible/prod-hosts prod-env-playbook.yml"'
                }
            }
        }
    }
}
