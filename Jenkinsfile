// This pipeline is for product microservice deployment
pipeline {
    agent {
        label 'java-slave'
    }

    // tools section 
    tools {
        maven 'maven-3.8.9'
        jdk 'JDK-17'
    }

    // environment 
    environment {
        // read the pom.xml file 
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        APPLICATION_NAME = "user"
        SONAR_URL = "http://34.30.23.85:9000"
        // SONAR_TOKEN = credentials('sonar_creds')

        DOCKER_HUB = "docker.io/i27devopsb7"
        DOCKER_CREDS = credentials('dockerhub_creds')
        //DOCKER_VM = "35.202.42.232"

    }

    // parameters
    parameters {
        choice(name: 'buildOnly',
            choices: 'no\nyes',
            description: 'This will only build the application'
        )
        choice(name: 'dockerPush',
            choices: 'no\nyes',
            description: 'This will only push the docker image to registry'
        )
        choice(name: 'deployToDev',
            choices: 'no\nyes',
            description: 'This will deploy the application to dev environment'
        )
        choice(name: 'deployToTest',
            choices: 'no\nyes',
            description: 'This will deploy the application to test environment'
        )
        choice(name: 'deployToStage',
            choices: 'no\nyes',
            description: 'This will deploy the application to stage environment'
        )
        choice(name: 'deployToProd',
            choices: 'no\nyes',
            description: 'This will deploy the application to prod environment'
        )
    }


    stages {
        stage ('Build'){
            when {
                anyOf {
                    expression { 
                        params.buildOnly == 'yes' 
                        params.dockerPush == 'yes' 
                    }
                }
            }
            steps {
                // using maven
                script {
                    buildApp().call()
                }
            }
        }
        // stage ('Sonar') {
        //     steps {
        //         echo "************** Starting Sonar Scan **************"
        //         withSonarQubeEnv('SonarQube') {
        //             sh """
        //                 mvn clean verify sonar:sonar \
        //                     -Dsonar.projectKey=i27-eureka \
        //                     -Dsonar.host.url=${env.SONAR_URL}\
        //                     -Dsonar.login=${env.SONAR_TOKEN}
        //             """
        //         }
        //         timeout (time: 2, unit: "MINUTES") {
        //             script {
        //                 waitForQualityGate abortPipeline: true
        //             }
        //         }
                 





        //     }
        // }
        // stage ('FormatBuild'){
        //     // existing i27-eureka-0.0.1-SNAPSHOT.jar
        //     // Destination i27-eureka-buildnumber-brnachname.jar
        //     steps {
        //         echo "Testing JAR Source: i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING}"
        //         echo "Testing JAR Destination: i27-${env.APPLICATION_NAME}-${BUILD_NUMBER}-${BRANCH_NAME}.${env.POM_PACKAGING}"
        //     }
        // }
        stage ('DockerBuildAndPush') {
            when {
                anyOf {
                    expression {
                        params.dockerPush == 'yes'
                    }
                }
            }
            steps {
                script {
                    dockerBuildAndPush().call()
                }
                // docker.io/i27devopsb7/eureka:v
            }
        } 
        stage ('Deploy To Dev') {
            when {
                anyOf {
                    expression { 
                        params.deployToDev == 'yes'
                    }
                }
            }
            steps {
                script {
                    // image validation
                    imageValidatiion().call()
                    dockerDeploy('dev', '5232').call()
                }
            }
        }
        stage ('Deploy To test') {
           when {
                anyOf {
                    expression { 
                        params.deployToTest == 'yes'
                    }
                }
           }
            steps {
                script{
                    // image validation
                    imageValidatiion().call()
                    dockerDeploy('test', '6232').call()
                }
            }
        }
        stage ('Deploy To Stage') {
            when {
                anyOf {
                    expression { 
                        params.deployToStage == 'yes'
                    }
                }
                anyOf {
                    branch 'release*'
                    tag pattern: "v\\d{1,2}\\.\\d{1,2}\\.\\d{1,2}", comparator: "REGEXP"
                }
            }
            steps {
                script{
                    // image validation
                    imageValidatiion().call()
                    dockerDeploy('stage', '7232').call()
                }
            }
        }
        stage ('Deploy To Prod') {

            // mailing implementaion

            when {
                anyOf {
                    expression { 
                        params.deployToProd == 'yes'
                    }
                }
                anyOf {
                    tag pattern: "v\\d{1,2}\\.\\d{1,2}\\.\\d{1,2}", comparator: "REGEXP"
                }
            }
            steps {
                timeout(time: 300, unit: 'SECONDS') {
                    input message: "Deploying ${env.APPLICATION_NAME} to production ?", ok: 'yes', submitter: 'i27academy, devlead'
                }
                
                script{
                    dockerDeploy('prod', '8232').call()
                }
            }
        }
    }
        
}


def buildApp() {
    return {
        echo "********** Building ${env.APPLICATION_NAME} Application *************"
        sh "mvn clean package -DskipTests=true" 
    }
}

def dockerDeploy(envDeploy, port){
    return {
        echo "Deploying to $envDeploy Environment"
            withCredentials([usernamePassword(credentialsId: 'john_docker_vm_creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                    // some block
                script {
                    try {
                        // stop the container
                        sh "sshpass -p $PASSWORD -v ssh -o StrictHostKeyChecking=no $USERNAME@$docker_vm_ip \"docker stop ${APPLICATION_NAME}-$envDeploy\""

                        // remove the container
                        sh "sshpass -p $PASSWORD -v ssh -o StrictHostKeyChecking=no $USERNAME@$docker_vm_ip \"docker rm ${APPLICATION_NAME}-$envDeploy\"" 
                    }
                    catch(err) {
                        echo "Error caught: $err"
                    }
                    // Creating a container
                    sh "sshpass -p $PASSWORD -v ssh -o StrictHostKeyChecking=no $USERNAME@$docker_vm_ip \"docker run --name ${APPLICATION_NAME}-$envDeploy -p $port:8132 -d ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:$GIT_COMMIT\""
                }
            } 
    }
}

def dockerBuildAndPush() {
    return {
        echo "************* Building the Docker image ***************"
        sh "cp target/i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} ./.cicd"
        sh "docker build --no-cache --build-arg JAR_SOURCE=i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:$GIT_COMMIT ./.cicd"
        echo "******************************************** Docker Login *********************************"
        sh "docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}"
        echo "******************************************** Docker Push *********************************"
        sh "docker push ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:$GIT_COMMIT"
    }
}

def imageValidatiion() {
    return {
        println ("***************************** Attempt to pull the docker image *********************")
        try {
            sh "docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:$GIT_COMMIT"
            println("********************** Image is Pulled Succesfully *************************")
        }
        catch(Exception e) {
            println("*************** OOPS, the docker image is not available...... So creating the image")
            buildApp().call()
            dockerBuildAndPush().call()

        }

    }
}

//  hostport:containerport

// dev > 5232:8232
// test > 6232:8232
// stage > 7232:8232
//prod > 8232:8232
