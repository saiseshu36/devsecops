pipeline {
  agent any
  tools { 
        maven 'Maven_3_8_4'  
    }
   stages{
    stage('CompileandRunSonarAnalysis') {
            steps {	
		sh 'mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=asgbuggywebapp0506_asgbuggywebapp -Dsonar.organization=asgbuggywebapp0506 -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=654ccd87d12a21532f8c26ebe933114f3180cdfe'
			}
    }
	stage('RunSCAAnalysisUsingSnyk') {
            steps {		
				withCredentials([string(credentialsId: 'Snyk_token', variable: 'SNYK_TOKEN')]) {
					sh 'mvn snyk:test -fn'
				}
			}
    }
	stage('Build') { 
            steps { 
               withDockerRegistry([credentialsId: "dockerlogin", url: ""]) {
                 script{
                 app =  docker.build("asg")
                 }
               }
            }
    }

	stage('Push') {
            steps {
                script{
                    docker.withRegistry('https://249085383813.dkr.ecr.us-east-1.amazonaws.com/asg', 'ecr:us-east-1:aws-credentials') {
                    app.push("latest")                                                                        
                    }
                }
            }
    	}
    stage('Configure EKS Cluster') {
            steps {
                withAWS(
                    credentials: 'aws-credentials',
                    region: 'us-east-1'
                ) {

                    sh '''
                        echo "Checking AWS Identity"
                        aws sts get-caller-identity
                        echo "Updating EKS Kubeconfig"
                        aws eks update-kubeconfig --name kubernetes-cluster --region us-east-1
                        echo "Current Kubernetes Context"
                        kubectl config current-context
                    '''
                }
            }
        }

	stage('Kubernetes Deployment of ASG Bugg Web Application') {
        steps {
            withAWS(credentials: 'aws-credentials', region: 'us-east-1') {

            sh 'kubectl get nodes'
            sh 'kubectl delete all --all -n devsecops || true'
            sh 'kubectl apply -f deployment.yaml --namespace=devsecops'
    }
  }
}
	   
	stage ('wait_for_testing'){
	   steps {
		   sh 'pwd; sleep 100; echo "Application Has been deployed on K8S"'
	   	}
	   }
	   
	stage('RunDASTUsingZAP') {
    steps {
        withAWS(credentials: 'aws-credentials', region: 'us-east-1') {

            sh '''
            pkill -f zap || true

            kubectl get nodes
            kubectl get svc -n devsecops

            HOST=$(kubectl get svc asgbuggy \
                -n devsecops \
                -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

            echo "HOST=$HOST"

            zap.sh -cmd \
                -port 8090 \
                -cmd \
                -quickurl http://$HOST:8002 \
                -quickprogress \
                -quickout ${WORKSPACE}/zap_report.html
            '''

            archiveArtifacts 'zap_report.html'
        }
      }
    }
  }
}
