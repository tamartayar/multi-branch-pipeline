pipeline {

    options {
        // Build auto timeout
        timeout(time: 60, unit: 'MINUTES')
    }

    // Some global default variables
    environment {
        IMAGE_NAME = 'website'
        TEST_LOCAL_PORT = 8817
        DEPLOY_PROD = false
        PARAMETERS_FILE = "${JENKINS_HOME}/parameters.groovy"
    }

    parameters {
        string (name: 'GIT_BRANCH',           defaultValue: 'master',  description: 'Git branch to build')
        booleanParam (name: 'DEPLOY_TO_PROD', defaultValue: false,     description: 'If build and tests are good, proceed and deploy to production without manual approval')


        // The commented out parameters are for optionally using them in the pipeline.
        // In this example, the parameters are loaded from file ${JENKINS_HOME}/parameters.groovy later in the pipeline.
        // The ${JENKINS_HOME}/parameters.groovy can be a mounted secrets file in your Jenkins container.
/*
        string (name: 'DOCKER_REG',       defaultValue: 'docker-artifactory.my',                   description: 'Docker registry')
        string (name: 'DOCKER_TAG',       defaultValue: 'dev',                                     description: 'Docker tag')
        string (name: 'DOCKER_USR',       defaultValue: 'admin',                                   description: 'Your helm repository user')
        string (name: 'DOCKER_PSW',       defaultValue: 'password',                                description: 'Your helm repository password')
        string (name: 'IMG_PULL_SECRET',  defaultValue: 'docker-reg-secret',                       description: 'The Kubernetes secret for the Docker registry (imagePullSecrets)')
        string (name: 'HELM_REPO',        defaultValue: 'https://artifactory.my/artifactory/helm', description: 'Your helm repository')
        string (name: 'HELM_USR',         defaultValue: 'admin',                                   description: 'Your helm repository user')
        string (name: 'HELM_PSW',         defaultValue: 'password',                                description: 'Your helm repository password')
*/
    }

    // In this example, all is built and run from the master
    agent { node { label 'master' } }

    // Pipeline stages
    stages {

        stage('Git clone and setup') {
            steps {
                echo "Check out website code"
                git branch: "master",
                        url: 'https://github.com/echovue/static_site.git'

                // Validate kubectl
//                sh "kubectl cluster-info"

                // Init helm client
 //               sh "helm init"

                // Make sure parameters file exists
                script {
                    if (! fileExists("${PARAMETERS_FILE}")) {
                        echo "ERROR: ${PARAMETERS_FILE} is missing!"
                    }
                }

                // Load Docker registry and Helm repository configurations from file
                load "${JENKINS_HOME}/parameters.groovy"

                echo "DOCKER_REG is ${DOCKER_REG}"
                echo "HELM_REPO  is ${HELM_REPO}"

                // Define a unique name for the tests container and helm release
                script {
                    branch = GIT_BRANCH.replaceAll('/', '-').replaceAll('\\*', '-')
                    ID = "${IMAGE_NAME}-${DOCKER_TAG}-${branch}"

                    echo "Global ID set to ${ID}"
                }
            }
        }

        stage('Build and tests') {
            steps {
                echo "Building application and Docker image"
                sh "${WORKSPACE}/build.sh --build --registry ${DOCKER_REG} --tag ${DOCKER_TAG} --docker_usr ${DOCKER_USR} --docker_psw ${DOCKER_PSW}"

                echo "Running tests"

                // Kill container in case there is a leftover
                sh "[ -z \"\$(docker ps -a | grep ${ID} 2>/dev/null)\" ] || docker rm -f ${ID}"

                echo "Starting ${IMAGE_NAME} container"
                sh "docker run --detach --name ${ID} --rm --publish ${TEST_LOCAL_PORT}:80 ${DOCKER_REG}/${IMAGE_NAME}:${DOCKER_TAG}"

                script {
                    host_ip = sh(returnStdout: true, script: '/sbin/ip route | awk \'/default/ { print $3 ":${TEST_LOCAL_PORT}" }\'')
                }
            }
        }

        // Run the 3 tests on the currently running Website Docker container
        stage('Local tests') {
            parallel {
                stage('Curl http_code') {
                    steps {
                        curlRun ("http://${host_ip}", 'http_code')
                    }
                }
                stage('Curl total_time') {
                    steps {
                        curlRun ("http://${host_ip}", 'total_time')
                    }
                }
                stage('Curl size_download') {
                    steps {
                        curlRun ("http://${host_ip}", 'size_download')
                    }
                }
            }
        }

        stage('Publish Docker and Helm') {
            steps {
                echo "Stop and remove container"
                sh "docker stop ${ID}"

                echo "Pushing ${DOCKER_REG}/${IMAGE_NAME}:${DOCKER_TAG} image to registry"
                sh "${WORKSPACE}/build.sh --push --registry ${DOCKER_REG} --tag ${DOCKER_TAG} --docker_usr ${DOCKER_USR} --docker_psw ${DOCKER_PSW}"

                echo "Packing helm chart"
                sh "${WORKSPACE}/build.sh --pack_helm --push_helm --helm_repo ${HELM_REPO} --helm_usr ${HELM_USR} --helm_psw ${HELM_PSW}"
            }
        }

        stage('Deploy to dev') {
            steps {
                script {
                    namespace = 'development'

                    echo "Deploying application ${ID} to ${namespace} namespace"
                    createNamespace (namespace)

                    // Remove release if exists
                    helmDelete (namespace, "${ID}")

                    // Deploy with helm
                    echo "Deploying"
                    helmInstall(namespace, "${ID}")
                }
            }
        }

        // Run the 3 tests on the deployed Kubernetes pod and service
        stage('Dev tests') {
            parallel {
                stage('Curl http_code') {
                    steps {
                        curlTest (namespace, 'http_code')
                    }
                }
                stage('Curl total_time') {
                    steps {
                        curlTest (namespace, 'time_total')
                    }
                }
                stage('Curl size_download') {
                    steps {
                        curlTest (namespace, 'size_download')
                    }
                }
            }
        }
    }
}
