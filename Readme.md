GitHub Actions Documentation
============================





## Componenets ##

[source documentation link
](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#the-components-of-github-actions)  

<!--- todo insert image https://docs.github.com/assets/cb-25535/images/help/images/overview-actions-simple.png --->

1. **Workflows:** consists of a number of jobs, will run triggered by an event, specified in `.github/workflows` directory.
2. **Events:** activity that triggers workflow run. e.g. pull req creation or push a commit.
3. **Jobs:** consists of a number of steps that execute on the same runner, can be a shell script or an action. jobs by default are not dependent to each other and run in parallel, but you can define **dependency**.
4. **Actions:** a frequent repeated task that you can reuse. writed by yourself or found in GitHub Marketplace.
5. **Runner:** is a server that runs your workflows when they're triggered. you can setup your own  runners.

----

## Self-hosted runners ##

> * Self-hosted runners can be physical, virtual, in a container

### Adding a self-hosted runner to a repository ###
follow this [link](https://docs.github.com/en/actions/hosting-your-own-runners/adding-self-hosted-runners#adding-a-self-hosted-runner-to-a-repository). this is not our requirment!

### Adding a self-hosted runner to an organization ###
[source documentation link](https://docs.github.com/en/actions/hosting-your-own-runners/adding-self-hosted-runners#adding-a-self-hosted-runner-to-an-organization)  

1. GitHub.com > main page of the organization > the left sidebar > Actions > Runners > New runner > New self-hosted runner
2. Select the operating system image and architecture and follow the instructions

    ```bash
    adduser actions-runner;
    # This user is non sudo user
    su - actions-runner;
    # In new user:
    mkdir actions-runner; cd actions-runner;
    curl -o actions-runner-linux-x64-2.299.1.tar.gz -L https://github.com/actions/runner/releases/download/v2.299.1/actions-runner-linux-x64-2.299.1.tar.gz;
    echo "147c14700c6cb997421b9a239c012197f11ea9854cd901ee88ead6fe73a72c74  actions-runner-linux-x64-2.299.1.tar.gz" | shasum -a 256 -c;
    tar xzf ./actions-runner-linux-x64-2.299.1.tar.gz
    # Configure
    ./config.sh --url https://github.com/<organization> --token <provided-token>
    # ./config.sh --unattended --name <arbitrary-name> --replace --labels <arbitrary-label-name> --url https://github.com/<organization> --token <provided-token>
    # ask for runner group
    # Enter the name of the runner group to add this runner to:
    # I prefer default
    ./run.sh 
    # For better deployment:
    # get back to root
    exit;
    cd /home/actions-runner/actions-runner/;
    /home/actions-runner/actions-runner/svc.sh install actions-runner;
    /home/actions-runner/actions-runner/svc.sh start;
    ```

---

## Create a Workflow ###

[source documentation link](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#create-an-example-workflow)

1. Add `.github/workflows` to your project.
```bash
cd <project-dir>;
mkdir -p .github/workflows;
```
2. create a file with arbitrary name with this content:
```bash
vim .github/workflows/deploy.yml
```
    name: Deploy
    run-name: ${{ github.actor }} is deploying!
    on: 
    push:
        branches:
        - 'main'
    jobs:
    update-service:
        runs-on: backup-server
        steps:
      - run: sudo /opt/scripts/frontend-update-service


- `name:` and `run-name:` are optional and just for showing in github actions page!
- `on:` specifies the trigger for this workflow. and here we say that when someone pushed on main branch, this workflow should run.
- `jobs:` groups together all the jobs
- `update-service:` is a job name that we specified!
- `runs-on:` specifies which runner should be used, we provide label of our self-hosted runner.
- `steps:` list of shell scripts or actions that will run. we run our update scripts here.

---

### Create scripts and access to run that script with sudo ###

1. make `/opt/scripts/` directory and write your scripts to update your service and give its permission to only root user (chmod 700 \</path/to/script\>).  

        #!/bin/bash

        LOG_FILE="/root/mosi.log"
        log () {
            echo "`date "+%Y-%m-%d %H:%M:%S"` ${1}" >> $LOG_FILE
        }

        if [ "$EUID" -ne 0 ]
        then 
            log "Please run as root";
            exit;
        fi
        
        log "mosi is updating sniproxy!"
        cd /opt/sni-proxy;
        log "+ git pull"
        git pull | tee -a $LOG_FILE;
        docker compose up -d --force-recreate | tee -a $LOG_FILE;
        log "mosi has updated sniproxy!"
        log

2. install `sudo` and give access to run just `update script` to `actions-runner`
```bash
apt install sudo
vim /etc/sudoers.d/deploy-scrpits-from-runners
# Add this line
actions-runner ALL=(root) NOPASSWD:/usr/bin/whoami,/opt/scripts/sniproxy-update
```

---

### Create and setup deploy keys ###

1. Create new key pair per repository
```bash
cd /root/.ssh;
ssh-keygen -t rsa -b 4096 -C "<your-email-address>";
# Enter name of key pair
# Leave passphrase empty
```

2. Add this public key to `Deploy keys` of your repo. Your repo > Settings > Deploy kes > Add deploy key

3. Add a config file to point project to ssh key
```bash
vim /root/.ssh/config
```
    Host prediction-frontend
        Hostname github.com
        IdentityFile ~/.ssh/prediction-frontend-deploy-key

4. Change git remote
```bash
git remote -v;
git remote set-url origin git@gitreponame:organization-name/repo-name.git
```
> `gitreponame` is what you specify as Host directive in ssh config. in this case is `prediction-frontend`.