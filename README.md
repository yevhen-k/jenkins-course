# Jenkins Couse

Based in the [playlist](https://www.youtube.com/playlist?list=PLg5SS_4L6LYvQbMrSuOjTL1HOiDhUE_5a)

## Jenkins on AWS

Flollow the tutorial in [Jenkins](https://www.jenkins.io/doc/tutorials/tutorial-for-installing-jenkins-on-AWS/) docs.

To check your public ip you can use:

```bash
curl http://checkip.amazonaws.com/
```

My EC2 instance is running on `eu-west-2` which is London.

Let's connect to the instance!

```bash
ssh -i "aws-jenkins-ec2-tests.pem" ubuntu@ec2-13-40-196-100.eu-west-2.compute.amazonaws.com
```

Installation of Java and [Jenkins](https://www.jenkins.io/doc/book/installing/linux/#debianubuntu):

```bash
sudo apt update
sudo apt install openjdk-11-jre
java -version
```

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

After that you'll be able to connect to the Jenkins via web (you'll have different address, but the same port):

```bash
http://ec2-13-40-196-100.eu-west-2.compute.amazonaws.com:8080
```

Then follow the tutorial.

## The First Jenkins Job

### Simple Job.
Select nothing below. Use 
`Build --> Execute Shell`
to make a hello world deploy app.

### Slow Down :)
You can use `sleep 5` to make it artificially slower

### Parallel Jobs
You can use `Execute concurrent builds if necessary` to start a few jobs simultaneously.

### Failed Jobs
To test how job may fail use command that will fail:
cat /etc/unexisting-file

### Keep It Clean
To keep Jenkins drive clean use `Discard old builds` with 2 possible policies:
- days to keep
- max # of builds to keep (usually up to 5)

You can install funny plugins on Jenkins: `ChuckNorris`

Check where builds are stored:
```bash
cd /var/lib/jenkins
ls -lah workspace/
ls -lah jobs/
cat Hello-World-Job-1/config.xml
cat Hello-World-Job-1/builds/3/log
```

## The Second Jenkins Job

We'll use buld with tests.

Use the following shell commands for build.

### Build

```bash
echo "------------- Build Started -------------"
cat <<EOF > index.html
<html>
<body bgcolor="black">
  <center>
    <h2>
      <font color="yellow">Hello Deploy!</font>
    </h2>
  </center>
</body>
</html>
EOF
echo "------------- Build Fineshed ------------"
```

### Test

```bash
echo "------------- Test Started --------------"
result=`grep "Hello" index.html | wc -l`
echo $result
if [ "$result" = "1" ]
then
  echo "Test passed"
  # exit 0
else
  echo "Test failed"
  exit 1
fi
echo "------------- Test Fineshed -------------"
```

Then check Jenkins `Workspace` to see some artifacts there.

Check if the script really tests something: replace, for example, `grep "Hello"` on `grep "something wrong"` and see what happened.

### Deploy

We're going to deploy via `ssh` with the use of plugin `Publish Over SSH`

Check files we're managed to build:

```bash
echo "------------- Deploy Started --------------"
ls -lah
echo "------------- Deploy Fineshed -------------"
```

You can even serve you `index.html` locally:

```bash
cd /var/lib/jenkins/workspace/Hello-World-Job-2-TEST
sudo python3 -m http.server 80
```

#### Start two new EC2 instances

`TEST-env` and `PROD-env`

Configure ssh-access on both of them (access via passphrase or publick key) as shown in github [tutorial](https://git-scm.com/book/en/v2/Git-on-the-Server-Generating-Your-SSH-Public-Key) and jenkins [tutorial](https://www.jenkins.io/doc/pipeline/steps/ssh-steps/):

This won't work because of [issue](https://issues.jenkins.io/browse/JENKINS-57495)
```bash
ssh-keygen -o
cat ~/.ssh/id_rsa
```

Use workaround instead. On the Jenkins machine generate RSA key:

```bash
ssh-keygen -t rsa -b 4096 -m PEM
cat ~/.ssh/id_rsa
```

Put the public part of the key (content of the `~/.ssh/id_rsa.pub` file) to the server you want to connect to (TEST-env or PROD-env) into `~.ssh/authorized_keys` file.

#### Configure Remote Servers

Make sure remote servers contain `/var/www/html` folder.

```bash
sudo mkdir -p /var/www/html
sudo chown -R ubuntu:ubuntu /var/www
```

Make sure you have `apache2` installed to run http service.

```bash
sudo apt install apache2
sudo systemctl start apache2
systemctl status apache2
```

Now you can visit you machine via public IP with your browser:
```bash
http://18.170.118.127/
```
You'll see the default apache web page.


#### Configure Jenkins to use SSH

Install `Publish Over SSH` plugin.

Go **Manage Jankins** --> **Configure System**. At the bottom of **Configure System** set up SSH settings.
Use private part of the generated earlier key here.

Privide:
- Name: PROD-env
- Hostname: ec2-3-8-94-141.eu-west-2.compute.amazonaws.com
- Username: ubuntu
- Remote Directory: /var/www/html


#### Configure Job to use SSH for Deployment.

**Post-build actions** --> **Send build artifacts over SSH**

- SSH-serner --> choose the server by name
- Transfers --> use `*` or `/*` to chose all files (`index.html` in our case)
- Exec command (in this particular case don't have, but as an example) --> `sudo systemctl restart apache2`

Go build and check how it works.
For debug purposes you may use:

**Send build artifacts over SSH** --> SSH server --> Advanced --> **Verbose output in console**

### Jenkins Slave/Nodes

- Make it easier for Jenkins
- More simultaneous builds
- Buils on different *platforms* on different servers
  - Node_1 for .NET
  - Node_2 for Java
  - Node_3 for C++

#### Preparation

- Add new node for Jankins Slave in the AWS
- Install Java:
  ```bash
  sudo apt update
  sudo apt install openjdk-11-jre
  java -version
  ```
- Install Docker: https://docs.docker.com/engine/install/ubuntu/
- Install `docker-compose` latest:
  ```bash
  sudo curl -L https://github.com/docker/compose/releases/download/v2.7.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
  sudo chmod +x /usr/local/bin/docker-compose
  docker-compose -v
  ```

We'll use SSH to make Master-Slave interactions.

1. Install `SSH Build Agents plugin` Jenkins plugin.

2. Configure Jenkins credentials. Manage Jenkins --> Manage Credentials --> Jenkins --> Global credentials (unrestricted) --> Add Credentials:
  - SSH Username with private key
  - Scope: System
  - Username: ubuntu
  - Private key -- enter directly. Here we're going to use PRIVATE RSA we've used earlier for connections to `TEST-env` and `PROD-env`.

2. Configure Jenkins Nodes (Slaves). Manage Jenkins --> Manage nodes and clouds --> New node:
  - Node name: Jenkins-slave CICD Tutorial
  - Permanent agent: selected
  - Number of executors: 4
  - Remote root directory: ./jenkins-agent
  - Labels: SLAVE-NODE
  - Launch method: Launch agents via SSH (from SSH [Build Agents plugin](https://plugins.jenkins.io/ssh-slaves/))
  - Host: ec2-18-133-219-181.eu-west-2.compute.amazonaws.com
  - Credentials: ubuntu (as we've created in *Configure Jenkins credentials.* paragraph)
  -  Host Key Verification Strategy: *Known hosts file Verification Strategy*. For the simplicity you can use *Manual trusted key Verification Strategy*


**Troubleshooting** for *Known hosts file Verification Strategy*:
1. You need first connect to the host (slave) manually from master node to update master's `.ssh/known_hosts` file)
2. Copy updated `.ssh/known_hosts` file to the master's Jenkins folder:

```bash
sudo mkdir /var/lib/jenkins/.ssh
sudo cp .ssh/known_hosts /var/lib/jenkins/.ssh/
sudo chown -R jenkins:jenkins /var/lib/jenkins/.ssh/
```

After that you are good to go to launch slave node and see `Agent successfully connected and online`!

#### Run Job on the Slave/Node

- Go to Job settings (`Configure`)
- General --> Restrict where this project can be run --> Label Expression (start typeing...)

## Stop And Start EC2

To imitate server restart and practice with AWS stop all your instances and start them again.

By default, AWS then give for your EC2 nodel new IP addresses. This may cause a few issues. To fix them you have to reconfigure Jenkins fo the new IP addresses.

You have update those IP addresses in the:
- Slave settings: Manaje Jenkins --> Manage nodes and clouds --> YOUR NODE --> Configure --> Host
- Jenkins: Manage Jenkins --> Configure System --> Publish over SSH --> Hostnames
- The IP of the webhook for GitHub may change after Jenkins server restart.

## Jenkins CLI Client

CLI Client makes you able to:
- Manage Jenkins from CLI
- Make skripts for Jenkins
- Manage Jenkins from console of your Jenkins server
- Manage Jenkins from you local console

Download Jenkins CLI:

1. With the use of command-line tools
```bash
wget ${JENKINS_URL}/jnlpJars/jenkins-cli.jar
```
2. From Jenkins UI
Jenkins --> Manage Jenkins --> Jenkins CLI (download by link)

### Experiments inside Jenkins Server

```bash
wget localhost:8080/jnlpJars/jenkins-cli.jar
```

Basic syntax example:

```bash
java -jar jenkins-cli.jar -auth username:password -s http://localhost:8080 who-am-i
```

```bash
java -jar jenkins-cli.jar -auth username:token -s http://localhost:8080 who-am-i
```

You should use **tokens** instead of **username:password**.

#### Create new user

**NB:** it is *optional* to create a user for CLI tasks. Here we just learn how to create new user and make a token to the user.

Manage Jenkins --> Manage Users --> Create User

#### Create token

First of all you have to log in to the Jenkins as a newly created user (if you created a new user)
Jenkin --> People --> SELECT USER --> Configure --> API Token

#### Run commands

```bash
java -jar jenkins-cli.jar -auth username:token -s http://localhost:8080 who-am-i
```

You can use environment variabeles to store `username` and `token`:

```bash
export JENKINS_USER_ID=user-cli
export JENKINS_API_TOKEN=1125a841abe8fb59f8eb3fbd1f268f7f75
```

```bash
java -jar jenkins-cli.jar -auth ${JENKINS_USER_ID}:${JENKINS_API_TOKEN} -s http://localhost:8080 who-am-i
```

And even shorter (in case you use exact `JENKINS_USER_ID` and `JENKINS_API_TOKEN` variabeles names)

```bash
java -jar jenkins-cli.jar -s http://localhost:8080 who-am-i
```

**NB**: variabeles created in this way will live *only* during the active session.

### Experiments from Your Machine

```bash
java -jar jenkins-cli.jar -auth user-cli:1125a841abe8fb59f8eb3fbd1f268f7f75 -s http://18.130.157.30:8080 who-am-i
```

### Back-up Jobs via CLI

Commands:
- `get-job`

```bash
java -jar jenkins-cli.jar -auth username:token -s http://18.130.157.30:8080 -webSocket get-job Hello-World-Job-2-PROD > Hello-World-Job-2-PROD.xml
```

- `create-job`
  
```bash
java -jar jenkins-cli.jar -auth username:token -s http://18.130.157.30:8080 -webSocket create-job CLI-Job-2-PROD < Hello-World-Job-2-PROD.xml
```

```bash
cat Hello-World-Job-2-PROD.xml | java -jar jenkins-cli.jar -auth username:token -s http://18.130.157.30:8080 -webSocket create-job CLI2-Job-2-PROD
```

- `delete-job`

```bash
java -jar jenkins-cli.jar -auth username:token -s http://18.130.157.30:8080 -webSocket delete-job CLI2-Job-2-PROD
```

## Jenkins & GitHub

### Semi-automated approach

*Prerequirement*: installed `GitHub plugin` plugin.

1. Create a dummy repo with nice html file `lorem-ipsum-html`
   - Make sure to create *private* repository. It is for educational purposes only. You can change the visibility of you project on our project's settings page accotding to [docs](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/managing-repository-settings/setting-repository-visibility)
   - get the public part of your ssh-key in the Jenkins Master Machine
      ```bash
      cat .ssh/id_rsa.pub
      ```
   - use this public part in your GitHub account 
   - test if you key is works. Try to clone you private repo inside Jenkins Master Machine
      ```bash
      git clone git@github.com:yevhen-k/lorem-ipsum-html.git
      ```


2. Add new SSH key to your GitHub account, give it a proper name, for example, `jenkins-aws-deploy`: https://github.com/settings/keys
   
3. Crete a new Job for cloning, testing, building, and deployment
   - Source Code Management: Git
   - Repository URL: git@github.com:yevhen-k/lorem-ipsum-html.git
   - Credentials: add SSH username with private key
   - Branch: master
   - Add build step
    ```bash
    echo "--- clone done ---"
    echo "--- check files ---"
    ls -lah
    head index.html
    echo "Build by Jenkins Build: $BUILD_ID" >> index.html
    echo "--- check files done ---"
    ```
   - Post-build Actions:  Send build artifacts over SSH
   - Name: `TEST-env`
4. Start Job
5. Check the results on your `TEST-env` service URL.

**NB**: this is semi-automated approach. You have to manually start build jobs.

### Job Build Triggers

In the Job Setting you can find the following build triggers:

1. Trigger scrips remotely (from script)
2. Build after other projects are built
3. Build periodically
4. Pool SCM (the same as *build periodically* but checks if new commit is in the repo)
5. Triggers from installed Plugins

#### Trigger scrips remotely (from script)

Let's make a new dummy build job `Remote-Build-Trigget-1`. And add some dummy echoes imitating some code execution.

- Use **Build Triggers** --> Trigger builds remotely (e.g., from scripts)
- Add some token, make it long and random
- Compose URL: JENKINS_URL/job/Remote-Build-Trigget-1/build?token=TOKEN_NAME

Sample of job content:
```bash
echo "Trigger scrips remotely (from script)"
echo $JOB_NAME
echo "Build by Jenkins Build: $BUILD_ID $JOB_NAME" >> index.html
echo "DONE"
```

```bash
http://18.130.157.30:8080/job/Remote-Build-Trigget-1/build?token=kfjpi4qp49askjf03up425p3jerf0923rjtzmnneadkgj984y382asljfhasnfciu3743
```

You can use this composed URL directly in your browser on which you are logged in to the Jenkins.

Another way to use the URL is with `username:password` or `username:token` credentials.

```bash
curl http://user-cli:1125a841abe8fb59f8eb3fbd1f268f7f75@18.130.157.30:8080/job/Remote-Build/build?token=kfjpi4qp49askjf03up425p3jerf0923rjtzmnneadkgj984y382asljfhasnfciu3743
```

#### Build after other projects are built

Let's make a new dummy build job `Remote-Build-Trigget-2`. And add some dummy echoes imitating some code execution.

- Use **Build Triggers** --> Build after other projects are built
- Specify what project builds you want track: `Remote-Build-Trigget-1`

Sample of job content:
```bash
echo "Build after other projects are built"
echo $JOB_NAME
echo "Build by Jenkins Build: $BUILD_ID $JOB_NAME" >> index.html
echo "DONE"
```

Start and check!

#### Build periodically

Let's make a new dummy build job `Remote-Build-Trigget-3`. And add some dummy echoes imitating some code execution.

- Use **Build Triggers** --> Build periodically
- `* * * * *` to start job every minute (cron-like syntax)
- Use **Disable project** to *not run it* :)

Sample of job content:
```bash
echo "Build periodically"
echo $JOB_NAME
echo "Build by Jenkins Build: $BUILD_ID $JOB_NAME" >> index.html
echo "DONE"
```

#### Pool SCM

(the same as *build periodically* but checks if new commit is in the repo)

Let's make a new dummy build job `Remote-Build-Trigget-4`. And attach remote repo to be cloned and processed.

- Use **Build Triggers** --> Poll SCM
- Use cron-like syntax to schedule the task: `* * * * *`
- Use **Source Code Management** to link to a GitHub repo
- Use **Build** --> Execute shell and add `echo "Build by Jenkins Build: $BUILD_ID $JOB_NAME" >> index.html` command (just to make it more interesting)
- Use **Post-build Actions** to send artifact to the `PROD-env`
- Make some changes to the repo and check if it works
- After you're done: don't forget to **Disable project** to *not run it* forever :)

#### Jenkins Job **Trigger from GitHub**

Let's make a new dummy build job `Remote-Build-Trigget-5`. And attach remote repo to be cloned and processed.

##### Configure Jenkins

To be able to use this functionality we have to install [GitHub Plugin](https://plugins.jenkins.io/github/).

- **Source Code Management**: Git. Add existing private repo with SSH-keys
- **Build**: Execute shell:
    ```bash
    echo "Build Triggered from GitHub"
    echo $JOB_NAME
    echo "Build by Jenkins Build: $BUILD_ID $JOB_NAME" >> index.html
    echo "DONE"
    ```
- **Post-build Actions**:  Send build artifacts over SSH. Send artifact to `PROD-env`
- **GitHub project**: URL of your project, the same as in **Source Code Management** step
- **Build Triggers**: GitHub hook trigger for GITScm polling

##### Configure GitHub

- Go to your project's GitHub page
- Navigate to **Setting**
- Select **Webhooks**, and add new one
  - **Payload URL**: http://JENKINS_URL:PORT/**github-webhook** (http://52.56.133.64:8080/github-webhook/). IMPORTANT: the IP of the webhook may change after Jenkins server restart.
  - **Content type**: application/json
  - **Which events would you like to trigger this webhook?**: Just the push event.

Push updates to the repo and check the result!


## Jobs With Parameters

Let's create a dummy Job which clones repo and does some things based on user's choice.

- **Source Code Management**: Git
- **Build Triggers**: None. We're going to make decisions manually
- **This project is parameterized**: add few params to learn the process
- **Build**: execute shell:
    ```bash
    ls -lah $FOLDERNAME
    echo "environment IS_PROD: $IS_PROD"
    echo "device: $DEVICE"
    ```

Save the Job. **Build with Parameters** option will appear on the left panel.


## Making Jenkins Back-up

Follow the [article](https://www.jenkins.io/doc/book/system-administration/backing-up/).



## Full CI/CD Pipeline

Configure Slave Node to be deployed on:

1. Install [Docker](https://docs.docker.com/engine/install/ubuntu/) 
2. Install `docker-compose` latest:
  ```bash
  sudo curl -L https://github.com/docker/compose/releases/download/v2.7.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
  sudo chmod +x /usr/local/bin/docker-compose
  docker-compose -v
  ```
Configure pipeline as in [Trigger from GitHub](####Jenkins-Job-**Trigger-from-GitHub**) section.

Use Slave Node as the node where the project is to be build. Follow the [Run Job on the Slave/Node](####run-job-on-the-slave/node) section.

**Build**: Execute shell:
```bash
echo "Build Triggered from GitHub"
sudo docker-compose down
sudo docker-compose build
sudo docker-compose up -d
echo "DONE"
```

### TODOs: Configure python web application:
- [x] docker
- [x] docker-compose
- [x] access to GPU
- [x] web hooks
- [ ] use build in a node with the following push to hub.docker.com and pull on the execution node
- [ ] blue-green deployment

See prepared example: https://github.com/yevhen-k/commonlitreadabilityprize-prjctr-service


## Troubleshooting

After instances restarted usually their IPs are changed.
You have update those IP addresses in the:
- Slave settings: Manage Jenkins --> Manage nodes and clouds --> YOUR NODE --> Configure --> Host
- Jenkins: Manage Jenkins --> Configure System --> Publish over SSH --> Hostnames
- The IP of the webhook for GitHub may change after Jenkins server restart.

**Troubleshooting** for *Known hosts file Verification Strategy*:
1. You need first connect to the host (slave) manually from master node to update master's `.ssh/known_hosts` file)
2. Copy updated `.ssh/known_hosts` file to the master's Jenkins folder:

```bash
sudo mkdir /var/lib/jenkins/.ssh
sudo cp .ssh/known_hosts /var/lib/jenkins/.ssh/
sudo chown -R jenkins:jenkins /var/lib/jenkins/.ssh/
```
