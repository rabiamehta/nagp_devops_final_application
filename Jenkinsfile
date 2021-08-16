pipeline{
    agent any

    tools{
        maven 'Maven3'
    }

    options{
        timestamps()
        timeout(time: 1, unit: 'HOURS')
    }

    environment{
        SONAR_PROJECT_NAME = 'rabiamehta-sonar'
        DOCKER_REPO_NAME = 'rabiamehta'
        USERNAME = 'rabia'
        DOCKER_FEATURE_PORT= 7400
        APP_PORT = 8080

    }

    stages{
        stage('Code Checkout'){
           steps{
               echo 'code checkout from feature branch'
               git branch: 'feature', url: "https://github.com/rabiamehta/nagp_devops_final_application.git"
           }
        }

        stage('Build'){
            steps{
                bat 'mvn clean install'
            }
        }

        stage('Unit Testing'){
            steps{
                bat 'mvn test'
            }
        }

        stage('sonar analysis'){
            steps{
                withSonarQubeEnv('SonarQubeScanner'){
                    bat "mvn sonar:sonar -Dsonar.projectName=${SONAR_PROJECT_NAME} -Dsonar.projectKey=${SONAR_PROJECT_NAME} -Dsonar.projectVersion=${BUILD_NUMBER}"
                }
                echo 'condition to fail if quality gates fails'
            }
        }

        stage('Build Docker Image'){
            steps{
                bat "docker build -t ${DOCKER_REPO_NAME}/i-${USERNAME}-${env.BRANCH_NAME}:${BUILD_NUMBER} -t ${DOCKER_REPO_NAME}/i-${USERNAME}-${env.BRANCH_NAME}:latest ."
            }
        }

        stage('Publish to DCR'){
            steps{
                withDockerRegistry([credentialsId: 'DockerHub', url: ""]){
                    bat "docker push ${DOCKER_REPO_NAME}/i-${USERNAME}-${env.BRANCH_NAME}:${BUILD_NUMBER}"
                    bat "docker push ${DOCKER_REPO_NAME}/i-${USERNAME}-${env.BRANCH_NAME}:latest "
                }
            }
        }

        stage('Deployment'){
            parallel{
                stage('Docker Deployment'){
                    steps{
                        script{
                            echo 'pre-container check'
                            containerIdCheck = "${bat (script: "docker ps -a -q -f status=running -f name=c-${USERNAME}-${env.BRANCH_NAME}", returnStdout: true).trim().readLines().drop(1).join("")}"
                            if(containerIdCheck != ''){
                                echo 'container is already running'
                                bat "docker stop c-${USERNAME}-${env.BRANCH_NAME}"
                                bat "docker rm c-${USERNAME}-${env.BRANCH_NAME}"
                            }else{
                                echo 'container is not running'
                            }

                            echo 'do docker deployment'
                            bat "docker run --name c-${USERNAME}-${env.BRANCH_NAME} -d -p ${DOCKER_FEATURE_PORT}:${APP_PORT} ${DOCKER_REPO_NAME}/i-${USERNAME}-${env.BRANCH_NAME}:latest"
                        }
                    }
                }
                stage('K8s Deployment'){
                    environment{
                        NS = ' nagp'
                        DEPLOYMENT_NAME = 'nagp-deployment-feature'
                    }
                    steps{
                        script{
                             echo 'k8s deployment status'
                             deploymentStatus = "${bat (script: "kubectl get deploy -n ${NS}")}"
                             if(deploymentStatus.contains("${DEPLOYMENT_NAME}")){
                                 echo ' deployment already exist'
                                 bat "kubectl rollout restart deployment/${DEPLOYMENT_NAME} -n ${NS}"
                             }else{
                                  echo 'fresh deployment'
                                  bat "kubectl apply -f k8s/"
                             }

                             echo 'wait for deployment status to succeed'
                             bat "kubectl rollout status -w deployment/${DEPLOYMENT_NAME} -n ${NS}"
                        }
                    }
                }
            }
        }
    }   

    post{
        success{
            echo 'Completed !'
        }
    }
}