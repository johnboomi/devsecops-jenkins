pipeline {
    agent any
    tools { 
        maven 'Maven_3_5_2'
    }
    stages {

        // New Stage for Setting Environment Variables
        stage('Set Environment Variables') {
            steps {
                withEnv(["FOO=bar", "BAZ=qux"]) {
                    sh 'echo FOO is $FOO'
                    sh 'echo BAZ is $BAZ'
                }
            }
        }

        stage('Compile and Run Sonar Analysis') {
            steps {
                script {
                    // Run Maven build and SonarQube analysis
                    sh 'mvn clean verify sonar:sonar ' +
                       '-Dsonar.projectKey=devops-project-demo-purposeku ' +
                       '-Dsonar.organization=ddevops-projectku ' +
                       '-Dsonar.host.url=https://sonarcloud.io ' +
                       '-Dsonar.token=54731ecc9ab46e7390da721fee672c4367279e4d'
                }
            }
        }

        stage('Run SCA Analysis Using Snyk') {
            steps {
                echo 'Running SCA Analysis with Snyk...'
                snykSecurity(
                    snykInstallation: 'synktool',
                    snykTokenId: 'synk_token',
                    failOnError: 'false',
                    failOnIssues: 'false'
                )
            }
            post {
                always {
                    echo 'SCA Analysis stage completed.'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                withDockerRegistry([credentialsId: 'docker', url: '']) {
                    script {
                        app = docker.build('secops')
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://590183914488.dkr.ecr.us-east-1.amazonaws.com/secops', 'ecr:us-east-1:aws-cred') {
                        app.push('latest')
                    }
                }
            }
        }

        stage('Kubernetes Deployment') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    script {
                        // Delete existing resources and apply new configuration
                        sh 'kubectl delete all --all -n devsecops'
                        sh 'kubectl apply -f deployment.yaml --namespace=devsecops'
                    }
                }
            }
        }

        stage('Wait for Deployment') {
            steps {
                sh 'sleep 180; echo "Application has been deployed on K8S"'
            }
        }

        stage('Run DAST Using ZAP') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    script {
                        // Run ZAP security scan
                        def serviceUrl = sh(script: 'kubectl get services/asgbuggy --namespace=devsecops -o json | jq -r ".status.loadBalancer.ingress[] | .hostname"', returnStdout: true).trim()
                        sh "zap.sh -cmd -quickurl http://${serviceUrl} -quickprogress -quickout ${WORKSPACE}/zap_report.html"
                        archiveArtifacts artifacts: 'zap_report.html'
                    }
                }
            }
        }
    }
}
