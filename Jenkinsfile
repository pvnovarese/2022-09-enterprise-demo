pipeline {
  environment {
    //
    // "REGISTRY" isn't required if we're using docker hub, I'm leaving it here in case you want to use a different registry
    // REGISTRY = 'registry.hub.docker.com'
    //
    // you need a credential named 'docker-hub' with your DockerID/password to push images
    CREDENTIAL = "docker-hub"
    DOCKER_HUB = credentials("$CREDENTIAL")
    REPOSITORY = "${DOCKER_HUB_USR}/${JOB_BASE_NAME}"
    //TAG = "build-${BUILD_NUMBER}"
    TAG = "${BRANCH_NAME}"
    IMAGELINE = "${REPOSITORY}:${TAG} Dockerfile"
    BRANCH_NAME = "${GIT_BRANCH.split("/")[1]}"
    //
    // we will need these if we're using anchore-cli
    // we'll need the anchore credential to pass the user
    // and password to anchore-cli so it can upload the results
    // ANCHORE_CREDENTIAL = "AnchoreJenkinsUser"
    // use credentials to set ANCHORE_USR and ANCHORE_PSW
    // ANCHORE = credentials("${ANCHORE_CREDENTIAL}")
    // api endpoint of your anchore instance
    // ANCHORE_URL = "http://anchore3-priv.novarese.net:8228/v1"
    //
  } // end environment 
  
  agent any
  
  stages {
    
    stage('Checkout SCM') {
      steps {
        checkout scm
      } // end steps
    } // end stage "checkout scm"
    
    stage('Verify Tools') {
      steps {
        sh """
          which docker
          #which anchore-cli
          #which /var/jenkins_home/anchorectl
          """
      } // end steps
    } // end stage "Verify Tools"

    
    stage('Build and Push Image') {
      steps {
        sh """
          echo ${DOCKER_HUB_PSW} | docker login -u ${DOCKER_HUB_USR} --password-stdin
          docker build -t ${REPOSITORY}:${TAG} --pull -f ./Dockerfile .
          docker push ${REPOSITORY}:${TAG}
        """
      } // end steps
    } // end stage "build and push"
    
    stage('Analyze Image with Anchore plugin') {
      steps {
        // anchore plugin for jenkins: https://www.jenkins.io/doc/pipeline/steps/anchore-container-scanner/
        // first, we need to write out the "anchore_images" file which is what the plugin reads to know
        // which images to scan:
        writeFile file: 'anchore_images', text: IMAGELINE
        // call the scanner, wrapin catchError so we can break the pipeline but still run the
        // cleanup stage if the evaluation fails
        catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
          // forceAnalyze is a good idea since we're passing a Dockerfile with the image
          anchore name: 'anchore_images', forceAnalyze: 'true', engineRetries: '900'
        }
        // if we want to use anchore-cli instead we can do this:
        // sh """
        //   anchore-cli --url ${ANCHORE_URL} --u ${ANCHORE_USR} --p ${ANCHORE_PSW} image add --force --dockerfile Dockerfile-1 --noautosubscribe ${REPOSITORY}:${TAG1}
        //   anchore-cli --url ${ANCHORE_URL} --u ${ANCHORE_USR} --p ${ANCHORE_PSW} image wait ${REPOSITORY}:${TAG1}
        //   anchore-cli --url ${ANCHORE_URL} --u ${ANCHORE_USR} --p ${ANCHORE_PSW} evaluate check ${REPOSITORY}:${TAG1}
        // """
        //
        // if you want continuous re-evaluation in the background, you can turn it on with these:
        //   anchore-cli subscription activate policy_eval ${REPOSITORY}:${TAG1}
        //   anchore-cli subscription activate vuln_update ${REPOSITORY}:${TAG1}
        // and in this case you would probably also want to configure "policy & vulnerability" updates
        // in "Events & Notifications" -> "Manage Notification Endpoints" 
        //
      } // end steps
    } // end stage "analyze image 1 with anchore plugin"     
    
    // optional, you could promote the image here but I need to figure out how
    // to skip this stage if the eval failed since I'm using catchError
    stage('Promote Image') {
      steps {
        sh """
          docker tag ${REPOSITORY}:${TAG} ${REPOSITORY}:${BRANCH_NAME}
          docker push ${REPOSITORY}:${BRANCH_NAME}
        """
      } // end steps
    } // end stage "Promote Image"        
    
    stage('Clean up') {
      steps {
        //
        // don't need the image(s) anymore so let's rm it
        //
        sh 'docker image rm ${REPOSITORY}:${TAG} ${REPOSITORY}:${BRANCH_NAME} || failure=1'
        // the || failure=1 just allows us to continue even if one or both of the tags we're
        // rm'ing doesn't exist (e.g. if the evaluation failed, we might end up here without 
        // re-tagging the image, so ${BRANCH_NAME} wouldn't exist.
        //
        // if we used anchore-cli above, we should probably use the plugin here to archive the evaluation
        // and generate the report:
        //anchore name: 'anchore_images', forceAnalyze: 'true', engineRetries: '900'        
      } // end steps
    } // end stage "clean up"
    
  } // end stages
  
} // end pipeline 
