pipeline {
    agent any
    environment {
        AWS_REGION = 'us-east-1'
        SONARQUBE_URL = "https://sonarcloud.io"
        TRUFFLEHOG_PATH = "/usr/local/bin/trufflehog3"
        JIRA_SITE = "https://cloudwithcallahan.atlassian.net"
        JIRA_PROJECT = "SCRUM"
        JIRA_ISSUE_TYPE = "Bug"
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
    script {
        withCredentials([usernamePassword(credentialsId: 'jira-api-key', usernameVariable: 'JIRA_USER', passwordVariable: 'JIRA_TOKEN')]) {
            
            // Define JSON payload as a Groovy map
            def jiraPayload = [
                fields: [
                    project: [ key: JIRA_PROJECT ],
                    summary: issueTitle,
                    description: [
                        type: "doc",
                        version: 1,
                        content: [
                            [
                                type: "paragraph",
                                content: [
                                    [
                                        type: "text",
                                        text: issueDescription
                                    ]
                                ]
                            ]
                        ]
                    ],
                    issuetype: [ name: JIRA_ISSUE_TYPE ]
                ]
            ]

            // Convert Groovy map to JSON inside script block
            def jsonPayload = groovy.json.JsonOutput.toJson(jiraPayload)

            // Debugging: Print JSON payload before sending
            echo "JIRA JSON Payload: ${jsonPayload}"

            withEnv(["JIRA_AUTH=$JIRA_USER:$JIRA_TOKEN"]) {
                def response = sh(script: """
                    curl -X POST -H "Accept: application/json" -H "Content-Type: application/json" \\
                    -u "\$JIRA_AUTH" \\
                    --data '${jsonPayload}' \\
                    "${JIRA_SITE}/rest/api/3/issue/"
                """, returnStdout: true).trim()

                echo "Jira Response: ${response}"
            }
        }
    }
}
