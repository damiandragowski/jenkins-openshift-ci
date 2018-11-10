//Jenkinsfile parameters
groupName = my-group
projectName = my-project
ftpServer = dev-ftp-1
deploySrv = dev-test-1
registryAddress = dev-reg-1

node('agents') {
    //Build parameters for builds and/or deployments, add more if necessary
    properties([
        choice(name: 'DoBuild', choices: 'true\nfalse', description: 'Build or deploy a previous image'),
        choice(name: 'WhichBuild', choices: (env.BUILD_NUMBER.toInteger()..1).join(\n), description: 'If deploying, which previous build'),
        choice(name: 'WhereToDeploy', choices: 'feature-branch\ndev-server\nopenshift-int\nopenshift-ppd\npre-prod-tar\nNONE', description: 'Where to deploy, NONE for no deploy')
    ])

    //extract build information ahead of time
    buildNumber = env.BUILD_NUMBER
    buildName = env.BRANCH_NAME + "/" + buildNumber
    buildToDeploy = params.DoBuild == "true" ? buildNumber : params.WhichBuild
    whereToDeploy = params.whereToDeploy
    buildType = params.DoBuild == "true" ? "build" : "deploy"
    branchName = env.BRANCH_NAME.replace("/", "-").replace("_", "-").toLowerCase() //replace problematic Openshift characters
    imageName = "${registryAddress}/${groupName}/${projectName}:${branchName}"
    buildDescription = (buildType == "build" ? "Build" : "Deployment from build ${buildNumber}") + " to ${whereToDeploy} started" 

    echo buildDescription
    //Optionally implement curl-based build started notification here using buildDescription, e.g Slack

    try {
        stage('clean') {
            //Clean tasks, for example:
            deleteDir()
        }

        stage('checkout') {
            //Checkout/clone/pull tasks, for example:
            checkout scm
        }

        if(buildType == "build") {
            stage('build') {
                //Build tasks, for example:
                sh "gradle bootRepackage"
            }

            stage('test') {
                //Test tasks, for example:
                try {
                    sh "gradle test"
                } finally {
                    junit 'build/test-results/test/*.xml' //fail build if JUnit plugin reports exceptions
                }
            }

            stage('docker') {
                //Build and push the image
                withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'docker-registry', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                    sh "sudo docker login -p $PASSWORD -u $USERNAME registry.address"
                }
                    docker.withRegistry('https://registry.address', 'docker-registry') {
                        sh "docker build -t ${imageName}-${buildNumber} ."
                        sh "docker push ${imageName}-${buildNumber}"
                    }
                }
            }
        }

        stage('deploy') {
            //If not deploying, finish
            if(whereToDeploy == "NONE")
                return

            if(whereToDeploy == "feature-branch" || whereToDeploy == "openshift-int" || whereToDeploy == "openshift-ppd") {
                //If deploying feature branch or int/preprod, first determine deployment name
                if(branchName == 'feature-development') deployName = "${projectName}-dev"
                if(whereToDeploy == 'openshift-int') deployName = "${projectName}-int"
                if(whereToDeploy == 'openshift-ppd') deployName = "${projectName}-ppd"
                else deployName = "${projectName}-${branchName}"

                //Then, login to Openshift, prepare the YAML files and execute oc commands
                withCredentials([string(credentialsId: 'registry-token', variable: 'TOKEN')]) {
                    sh "oc login ${registry-address} --token==$TOKEN"
                }

                sh "sed -i 's/BRANCHNAME/${deployName}/g' deploy/deployment.yml"
                sh "sed -i 's/BRANCHNAME/${deployName}/g' deploy/service.yml"
                sh "sed -i 's/BRANCHNAME/${deployName}/g' deploy/route.yml"
                sh "sed -i 's/DEPLOYNAME/${deployName}/g' deploy/deployment.yml"
                sh "sed -i 's/DEPLOYNAME/${deployName}/g' deploy/service.yml"
                sh "sed -i 's/DEPLOYNAME/${deployName}/g' deploy/route.yml"

                sh "oc project ${groupName}"
                sh "oc apply -f deploy/deployment.yml"
                sh "oc apply -f deploy/service.yml"
                sh "oc apply -f deploy/route.yml"

                //Finally, login and push the image to the registry
                docker.withRegistry('https://registry.address', 'docker-registry') {
                    sh "docker pull ${imageName}-${buildToDeploy}"
                    sh "docker tag ${imageName}-${buildToDeploy} ${imageName}"
                    sh "docker push ${imageName}" //Openshift will use the image tagged :latest
                }
            }
            else if(whereToDeploy == 'dev-server') {
                //If deploying to a simple server, use SSH to pull and run the image, then wait for healthCheck to resolve
                withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'docker-registry', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'server-login', usernameVariable: 'SRVUSERNAME', passwordVariable: 'SRVPASSWORD']]) {
                        sh """
                            sshpass -p${SRVPASSWORD} ssh -o StrictKeyChecking=no -o UserKnownHostsFile=/dev/null ${SRVUSERNAME}@${deploySrv} '\
                            docker login p ${PASSWORD} -u ${USERNAME} ${registryAddress}; \
                            docker system prune -f; docker kill ${projectName}; docker rm ${projectName}; \
                            docker pull ${imageName}-${buildToDeploy}; \
                            docker run --detach --name ${projectName} --hostname=\$HOSTNAME ${imageName}-${buildToDeploy}';
                            bash -c 'until [[ "\$curl -s --output /dev/null --head --fail -w ''%{http_code}'' \
                            http://${whereToDeploy}/healthCheck )" = 200 ]] ; do echo '.'; sleep 5; done; \
                            echo 'Server is up';
                        """
                    }
                }
            }
            else if(whereToDeploy == 'pre-prod-tar') {
                //If deploying a TAR for production, pull, tag, save and upload to an FTP server
                sh """
                    docker login p ${PASSWORD} -u ${USERNAME} ${registryAddress}; \
                    docker pull ${imageName}-${buildToDeploy}; \
                    docker tag ${imageName}-${buildToDeploy} ${groupName}:${projectName}-${buildToDeploy}; \
                    docker save ${groupName}:${projectName}-${buildToDeploy} ${groupName}:${projectName}-${buildToDeploy}.tar; \
                    curl -T ${groupName}:${projectName}-${buildToDeploy}.tar ftp://${ftpServer};
                """
            }
        }

        currentBuild.result = 'SUCCESS'
    }
    catch(e) {
        throw e;
    }
    finally {
        resultString = currentBuild.result == 'SUCCESS' ? "successful" : "failed"
        resultMessage = "${projectName} build ${buildName} ${resultString}"

        //Optionally implement curl-based build finished notification here using resultMessage
    }
}