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

## Part 2: Get Syft and Grype
(optional, the Jenkinsfile only uses anchore-cli but we can split some of this stuff out using Anchore toolbox)
We can download the binaries directly into our bind mount directory we created we spun up the jenkins container:

`curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sudo sh -s -- -b /tmp/jenkins-data`
`curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sudo sh -s -- -b /tmp/jenkins-data`

## STOP READING HERE
(I haven't fixed the rest of this yet)


## Part 3: Check for CVEs with Grype

- Fork this repo
- In the Jenkinsfile, change this line, replacing “pvnovarese” with your Docker ID:
	`repository = 'pvnovarese/jenkins-grype-demo'`
- From the jenkins main page, select “New Item” 
- Name it “jenkins-grype-demo”
- Choose “pipeline” and click “OK”
- On the configuration page, scroll down to “Pipeline”
- For “Definition,” select “Pipeline script from SCM”
- For “SCM,” select “git”
- For “Repository URL,” paste in the URL of your forked github repo
	e.g. https://github.com/pvnovarese/jenkins-grype-demo (use your github username)
- Click “Save”
- You’ll now be at the top-level project page.  Click “Build Now”

Jenkins will check out the repo and build an image using the provided Dockerfile.  This image will be a simple copy of dvwa (dang vulnerable web app), which is an example app that is full of known vulnerabilities.  Once the image is built, Jenkins will call grype and then grep through the output to search for High and Critical issues.  This should cause the pipeline to fail at the “Analyze with grype” stage.

You can check the console output for the build if you want to see where the failure occurs.

If you’d like to see a successful build, go to the github repo, edit the Dockerfile, and change the FROM image from dvwa to something like alpine:latest, then go back to the Jenkins project page and click “Build now” again. This time, once the image passes our grype check, jenkins will rebuild the image, using the “prod” tag this time, and push it to Docker Hub using the credentials you provided.

## Part 4: Package Stoplist with Syft (optional)
There is a companion repo and demo for Anchore Syft here: https://github.com/pvnovarese/jenkins-syft-demo

**Challenge: can you make a single Jenkinsfile that will pass an image through both syft and grype?**

## Part 5: Cleanup
- Kill the jenkins container (it will automatically be removed since we specified --rm when we created it):
	`pvn@gyarados /home/pvn> docker kill jenkins`
- Remove the jenkins-data directory from /tmp
	`pvn@gyarados /home/pvn> sudo rm -rf /tmp/jenkins-data/`
- Remove all demo images from your local machine:
	`pvn@gyarados /home/pvn> docker image ls | grep -E "jenkins-grype-demo|jenkins-syft-demo" | awk '{print $3}' | xargs docker image rm -f`

