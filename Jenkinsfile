pipeline {
    agent any
    tools { 
        maven 'Maven_3_5_2'  
    }
    stages {
        stage('CompileandRunSonarAnalysis') {
            steps {	
                sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=devsecops28_devsecops-demo -Dsonar.organization=devsecops28 -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=3d75b426c597d81980065bcf2087657ff5306998'
            }
        }

        stage('RunSCAAnalysisUsingSnyk') {
            steps {		
                echo 'Testing...'
                snykSecurity(
                    snykInstallation: 'synktool',
                    snykTokenId: 'synk_token',
                    failOnError: 'false',
                    failOnIssues: 'false'
                )
            }
        }

        stage('Build') { 
            steps { 
                withDockerRegistry([credentialsId: "docker", url: ""]) {
                    script {
                        app = docker.build("secops")
                    }
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    docker.withRegistry('https://056296809003.dkr.ecr.us-east-2.amazonaws.com', 'ecr:us-east-2:aws-cred') {
                        app.push("latest")
                    }
                }
            }
        }

        stage('Kubernetes Deployment of secops Bugg Web Application') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    sh('kubectl delete all --all -n devsecops')
                    sh('kubectl apply -f deployment.yaml --namespace=devsecops')
                }
            }
        }

        stage('wait_for_testing') {
            steps {
                sh 'pwd; sleep 180; echo "Application Has been deployed on K8S"'
            }
        }
        
        stage('RunDASTUsingZAP') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    sh('zap.sh -cmd -quickurl http://$(kubectl get services/asgbuggy --namespace=devsecops -o json| jq -r ".status.loadBalancer.ingress[] | .hostname") -quickprogress -quickout ${WORKSPACE}/zap_report.html')
                    archiveArtifacts artifacts: 'zap_report.html'
                }
            }
        } 
    }
}
