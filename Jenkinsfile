node {
    checkout scm
    withEnv(['HOME=.']) {          
        docker.image('hello-edge:v1.0.0').withRun(""" --privileged  """) { c ->
            // Remove docker.withRegistry since we will use manual login
            docker.image('$DOCKER_IMAGE_CLI').inside(""" --link ${c.id}:docker --privileged -u root """) {
                
                stage ('Build') {
                    sh """
                        cd app
                        echo "$DOCKER_IMAGE_CLI"
                        docker-compose --host tcp://docker:2375 build
                        docker --host tcp://docker:2375 images
                        cd ..
                    """
                }
                
                stage ('Upload') {
                    // Use Docker Hub credentials for login
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-token', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh """
                            cp -RT app /app/src/workspace
                            cd /app/src/workspace

                            export IE_SKIP_CERTIFICATE=true
                            export EDGE_SKIP_TLS=1

                            iectl config add publisher --name "publisherdev" --dockerurl "http://127.0.0.1:2375" --workspace "/app/src/workspace"
                            iectl publisher workspace init
                            iectl publisher docker-engine v -u http://localhost:2375
                            
                            iectl config add iem --name "iemdev" --url $IEM_URL --user $USER_NAME --password $PSWD

                            iectl publisher standalone-app create --reponame $REPO_NAME --appdescription "uploaded using Jenkins" --iconpath $ICON_PATH --appname $APP_NAME

                            def version = sh(script: "iectl publisher standalone-app version list -a ${APP_NAME} -k 'versionNumber' | python3 ./getAppVersion.py", returnStdout: true).trim()

                            def version_new = sh(script: "echo ${version} | awk -F. -v OFS=. 'NF==1{print ++\$NF}; NF>1{if(length(\$NF+1)>length(\$NF))\$(NF-1)++; \$NF=sprintf(\"%0*d\", length(\$NF), (\$NF+1)%(10^length(\$NF))); print}'", returnStdout: true).trim()

                            echo 'new Version: '$version_new

                            // Docker login using credentials
                            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

                            // Tag and push Docker image
                            docker tag $DOCKER_IMAGE_CLI $DOCKER_USERNAME/$REPO_NAME:$version_new
                            docker push $DOCKER_USERNAME/$REPO_NAME:$version_new
                            
                            // Docker logout
                            docker logout

                            // Final steps for iectl
                            iectl publisher standalone-app version create --appname $APP_NAME --changelogs "new release" --yamlpath "./app/docker-compose.prod.yml" --versionnumber $version_new 
                            -n '{"hello-edge":[{"name":"hello-edge","protocol":"HTTP","port":"80","headers":"","rewriteTarget":"/"}]}' -s 'hello-edge' -t 'FromBoxReverseProxy' -u "hello-edge" -r "/"

                            iectl publisher app-project upload catalog --appname $APP_NAME -v $version_new
                        """
                    }
                }
            }        
        }
    }
}
