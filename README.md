# Demo: Integrating Anchore, Jenkins, and Jira (for Policy Violations)

This is a very rough demo of integrating Jenkins, Anchore, and Jira.  This repo will open Jira tickets for violations of policy detected by Anchore Enterprise in the images we build and scan.

## Part 1: Jenkins Setup

We're going to run jenkins in a container to make this fairly self-contained and easily disposable.  This command will run jenkins and bind to the host's docker sock (if you don't know what that means, don't worry about it, it's not important).

`$ docker run -u root -d --name jenkins --rm -p 8080:8080 -p 50000:50000 -v /var/run/docker.sock:/var/run/docker.sock -v /tmp/jenkins-data:/var/jenkins_home jenkinsci/blueocean
`

and we'll need to install jq, python3, and anchore-cli in the jenkins container:

`$ docker exec jenkins apk add jq python3 && python3 -m ensurepip && pip3 install anchore-cli`

Once Jenkins is up and running, we have just a few things to configure:
- Get the initial password (`$ docker logs jenkins`)
- log in on port 8080
- Unlock Jenkins using the password from the logs
- Select “Install Selected Plugins” and create an admin user
- Create a credential so we can push images into Docker Hub:
	- go to manage jenkins -> manage credentials
	- click “global” and “add credentials”
	- Use your Docker Hub username and password (get an access token from Docker Hub if you are using multifactor authentication), and set the ID of the credential to “Docker Hub”.

## Part 2: Get Syft and Grype (Optional)
(optional, the Jenkinsfile only uses anchore-cli but we can split some of this stuff out using Anchore toolbox)
We can download the binaries directly into our bind mount directory we created we spun up the jenkins container:

`curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sudo sh -s -- -b /tmp/jenkins-data`
`curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sudo sh -s -- -b /tmp/jenkins-data`


## Part 3: Check for fixable CVEs and open Jira tickets automatically

- Fork this repo
- In the Jenkinsfile, change the lines in the Environment section as needed to reflect your credentials and Anchore/Jira endpoints, project IDs, etc.
- From the jenkins main page, select “New Item” 
- Name it “jenkins-anchore-jira-policy”
- Choose “pipeline” and click “OK”
- On the configuration page, scroll down to “Pipeline”
- For “Definition,” select “Pipeline script from SCM”
- For “SCM,” select “git”
- For “Repository URL,” paste in the URL of your forked github repo
	e.g. https://github.com/pvnovarese/jenkins-anchore-jira-policy (use your github username)
- Click “Save”
- You’ll now be at the top-level project page.  Click “Build Now”

Jenkins will check out the repo and build an image using the provided Dockerfile.  This image is based on ubuntu with a sudo package installed that has known vulnerabilties.  Once the image is built, Jenkins will add it to Anchore's queue to be analyzed, pull the vulnerability report, and then extract the policy evaluation report.  If the image has a final action of STOP (and that action is because of the policy evaluation, not from the image being blacklisted etc), it will extract the list of individual stop actions from the report and open a Jira ticket with the the Anchore Trigger ID and Check Output for each violation (this could be adjusted to include WARN actions if you want).

The output you get will depend on the policy bundle you have set as the default.  If you have a particular image that you want to test with, you can edit the Jenkinsfile to skip the "Build image and tag with build number" stage and just manually specify the image you want in the REPOSITORY and TAG variables in the Environment section.

## Part 4: Cleanup
- Kill the jenkins container (it will automatically be removed since we specified --rm when we created it):
	`pvn@gyarados /home/pvn> docker kill jenkins`
- Remove the jenkins-data directory from /tmp
	`pvn@gyarados /home/pvn> sudo rm -rf /tmp/jenkins-data/`
- Remove all demo images from your local machine:
	`pvn@gyarados /home/pvn> docker image ls | grep -E "jenkins-anchore-jira-policy" | awk '{print $3}' | xargs docker image rm -f`

