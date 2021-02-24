pipeline {
  environment {
    // shouldn't need the registry variable unless you're not using dockerhub
    // registry = 'registry.hub.docker.com'
    //
    // change this HUB_CREDENTIAL to the ID of whatever jenkins credential has your registry user/pass
    // first let's set the docker hub credential and extract user/pass
    // we'll use the USR part for figuring out where are repository is
    HUB_CREDENTIAL = "docker-hub"
    // use credentials to set DOCKER_HUB_USR and DOCKER_HUB_PSW
    DOCKER_HUB = credentials("${HUB_CREDENTIAL}")
    
    // we'll need the anchore credential to pass the user
    // and password to syft so it can upload the results
    ANCHORE_CREDENTIAL = "AnchoreJenkinsUser"
    // use credentials to set ANCHORE_USR and ANCHORE_PSW
    ANCHORE = credentials("${ANCHORE_CREDENTIAL}")
    
    // url for anchore-cli
    ANCHORE_CLI_URL = "http://anchore-priv.novarese.net:8228/v1/"
    
    // use credentials to set JIRA_USR and JIRA_PSW
    JIRA_CREDENTIAL = "jira-anchore8"
    JIRA = credentials("${JIRA_CREDENTIAL}")
    JIRA_URL = "anchore8.atlassian.net"
    
    JIRA_PROJECT = "10000"
    JIRA_ISSUETYPE = "10002"
    JIRA_ASSIGNEE = "5fc52f03f2df6c0076c94c94"
    
    // change repository to your DockerID
    REPOSITORY = "${DOCKER_HUB_USR}/jenkins-anchore-jira-policy"
    TAG = ":devbuild-${BUILD_NUMBER}"   
    //TAG = ":scratch1"
    
    // set path for executables.  I put these in jenkins_home as noted
    // in README but you may install it somewhere else like /usr/local/bin
    SYFT_LOCATION = "/var/jenkins_home/syft"
    GRYPE_LOCATION = "/var/jenkins_home/grype"
  } // end environment
  
  agent any
  stages {
    
    stage('Checkout SCM') {
      steps {
        checkout scm
      } // end steps
    } // end stage "checkout scm"
    
    stage('Build image and tag with build number') {
      steps {
        script {
          dockerImage = docker.build REPOSITORY + TAG
          docker.withRegistry( '', HUB_CREDENTIAL ) { 
            dockerImage.push() 
          }
        } // end script
      } // end steps      
    } // end stage "build image and tag w build number"
    
    stage('Analyze with Anchore') {
      steps {
        //
        // anlayze image, get the evaluation, and parse it for stop actions
        // we will only open tickets for images that have final action = "stop
        // AND reason = "policy evaluation" (i.e. we won't open a ticket if the 
        // image fails because it's on a image blocklist)
        // we then select the policy actions that have a "stop" gate action AND
        // are not whitelisted, then output the trigger ID and check output.
        //
        sh """
          echo "scanning ${REPOSITORY}:${TAG}"
          ## queue image for analysis
          anchore-cli --url ${ANCHORE_CLI_URL} --u ${ANCHORE_USR} --p ${ANCHORE_PSW} image add ${REPOSITORY}${TAG}
          ## wait for analysis to complete
          anchore-cli --url ${ANCHORE_CLI_URL} --u ${ANCHORE_USR} --p ${ANCHORE_PSW} image wait ${REPOSITORY}${TAG}
          ## get the evaluation and wash it through jq.
          anchore-cli --url ${ANCHORE_CLI_URL} --u ${ANCHORE_USR} --p ${ANCHORE_PSW} --json evaluate check --detail ${REPOSITORY}${TAG} | \
            jq -r '.[keys_unsorted[0]] | .[keys_unsorted[0]] | .[keys_unsorted[0]] | .[].detail.result | 
            select ((.final_action=="stop") and (.final_action_reason=="policy_evaluation")) | .result[].result?.rows[] | 
            select (.[6]=="stop") | [.[2], .[5]] | @tsv' > jira_body.txt
            ##
            ## this is an extremely ugly jq filter, but I'll try to sort it out:
            ##
            ## Line 1:
            ## the keys_unsorted functions are necessary because we don't know what the keys will
            ## be, exactly (the first key we have to contend with is the digest, the second is the image
            ## registry/repo:tag.  Once we get past that we can dive down into detail.result
            ##
            ## Line 2:
            ## only proceed if the final_action is stop AND final_action_reason is policy_evaluation
            ## (e.g. we won't do anything if the image is blocklisted (this is a personal preference,
            ## there is a good argument that stop actions in a blocklisted image should still be
            ## called out and addressed, I just did it this way to illustrate a possibility)).
            ## Once we clear that hurdle, filter down to the result 
            ## 
            ## Line 3:
            ## only select rows where Gate_Action (index 6) is "Stop" and for those, output index 2 
            ## (Trigger_Id) and 5 (Check_Output), and format them as tab seperated values.
            ##
            ## warning: I am also a jq noob so there's probably a better way to do that 
            ##
        """
      } // end steps
    } // end stage "analyze"

     stage('Open Jira Ticket if Needed') {
      steps {       
        script {
          DESC_BODY_LINES = sh (
            script: 'cat xxx_jira_body.txt | wc -l',
            returnStdout: true
          ).trim()
          if (DESC_BODY_LINES != '0') {
            sh """
              # build json for jira API
              echo '{ "fields": { "project": { "id": "${JIRA_PROJECT}" }, "issuetype": { "id": "${JIRA_ISSUETYPE}" }, "summary": "Anchore detected policy violations", "reporter": { "id": "${JIRA_ASSIGNEE}" }, "labels": [ "anchore" ], "assignee": { "id": "${JIRA_ASSIGNEE}" }, "description": "' | head -c -1 > xxx_jira_header.txt
              echo '${REPOSITORY}${TAG} has STOP action policy violations:' >> xxx_jira_header.txt
              echo >> xxx_jira_header.txt
              cat xxx_jira_header.txt xxx_jira_body.txt | sed -e :a -e '\$!N;s/\\n/\\\\n/;ta' | tr '\\t' '  ' | tr -d '\\\n' > xxx_jira_v2_payload.json  # escape newlines, convert tabs to spaces, remove any remaining newlines
              echo '" } }' >> xxx_jira_v2_payload.json # this just closes the json
              echo "opening jira ticket"
              cat xxx_jira_v2_payload.json | curl --data-binary @- --request POST --url 'https://${JIRA_URL}/rest/api/2/issue' --user '${JIRA_USR}:${JIRA_PSW}'  --header 'Accept: application/json' --header 'Content-Type: application/json'
            """
          } else {
            echo "no problems detected"
          } //end if/else
        } //end script
        
      } // end steps
    } // end stage "Open Jira"
    
    
    //stage('Re-tag as prod and push stable image to registry') {
    //  steps {
    //    script {
    //      docker.withRegistry('', HUB_CREDENTIAL) {
    //        dockerImage.push('prod') 
    //        // dockerImage.push takes the argument as a new tag for the image before pushing
    //      }
    //    } // end script
    //  } // end steps
    //} // end stage "retag as prod"

    stage('Clean up') {
      // delete the images locally
      steps {
        sh 'docker rmi ${REPOSITORY}${TAG}'
        // ${REPOSITORY}:prod'
      } // end steps
    } // end stage "clean up"

  } // end stages
} // end pipeline
