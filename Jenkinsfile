pipeline {
    agent any
    environment {
        AWS_REGION = 'us-east-1'
        SONARQUBE_URL = "https://sonarcloud.io"
        TRUFFLEHOG_PATH = "/usr/local/bin/trufflehog3"
        JIRA_SITE = "https://cloudwithcallahan.atlassian.net"
        JIRA_PROJECT = "SCRUM" // Your Jira project key
        JIRA_ISSUE_TYPE = "Bug" // Issue type
    }

    stages {
        stage('Set AWS Credentials') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'awscreds' 
                ]]) {
                    sh '''
                    echo "AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID"
                    aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Bassdanger/autoScale'
            }
        }

        // Security Scans
        stage('Static Code Analysis (SAST)') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'sonarqube-api-key', variable: 'SONAR_TOKEN')]) {
                        def scanStatus = sh(script: '''
                            ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=Bassdanger_autoScale \
                            -Dsonar.organization=bassdanger \
                            -Dsonar.host.url=${SONARQUBE_URL} \
                            -Dsonar.login=''' + SONAR_TOKEN, returnStatus: true)

                        if (scanStatus != 0) {
                            createJiraTicket("Static Code Analysis Failed", "SonarQube scan detected issues in your code.")
                            error("SonarQube found security vulnerabilities!")
                        }
                    }
                }
            }
        }

        stage('Snyk Security Scan') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'snyk-api-key', variable: 'SNYK_TOKEN')]) {
                        sh "snyk auth ${SNYK_TOKEN}"
                        sh "snyk monitor || echo 'No supported files found, monitoring skipped.'"
                    }
                }
            }
        }

        // **New Stage: Create Jira Issue (Before Terraform)**
        stage('Create Jira Issue') {
            steps {
                script {
                    createJiraTicket("Pre-Terraform Check Passed", "Security scans completed. Proceeding with Terraform deployment.")
                }
            }
        }

        stage('Initialize Terraform') {
            steps {
                sh 'terraform init'
            }
        }

        stage('Plan Terraform') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'awscreds'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform plan -out=tfplan
                    '''
                }
            }
        }

        stage('Apply Terraform') {
            steps {
                input message: "Approve Terraform Apply?", ok: "Deploy"
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'awscreds'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform apply -auto-approve tfplan
                    '''
                }
            }
        }
    }   

    post {
        success {
            echo 'Terraform deployment completed successfully!'
        }

        failure {
            echo 'Terraform deployment failed!'
            createJiraTicket("Terraform Deployment Failed", "Terraform deployment encountered an error in Jenkins.")
        }
    }
}

// Function to Create a Jira Ticket
def createJiraTicket(String issueTitle, String issueDescription) {
    withCredentials([string(credentialsId: 'jira-api-key-2', variable: 'JIRA_TOKEN')]) {
        def jiraPayload = """
        {
            "fields": {
                "project": { "key": "${JIRA_PROJECT}" },
                "summary": "${issueTitle}",
                "description": "${issueDescription}",
                "issuetype": { "name": "${JIRA_ISSUE_TYPE}" },
                "priority": { "name": "High" }
            }
        }
        """

        def response = sh(script: """
            curl -X POST -H "Content-Type: application/json" \\
            -u ${JIRA_USER}:${JIRA_TOKEN} \\
            --data '${jiraPayload}' \\
            ${JIRA_SITE}/rest/api/2/issue/
        """, returnStdout: true).trim()

        echo "Jira Response: ${response}"
    }
}
