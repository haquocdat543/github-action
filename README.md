# github action
This is a demonstration of using github-action

## Prerequisites
* [git](https://git-scm.com/downloads)
* [terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
* [awscli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [config-profile](https://docs.aws.amazon.com/cli/latest/reference/configure/)
## Start
### 1. Clone project
```
git clone https://github.com/haquocdat543/action.git
cd action
```
### 2. Initialize
```
cd backend
terraform init
terraform apply --auto-approve
```
```
Initializing the backend...
Initializing provider plugins...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Using previously-installed hashicorp/aws v5.30.0
Terraform has been successfully initialized!
You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.
If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
var.key_pair
  Enter a value:
```
Enter your `key pair` name. If you dont have create one. [keypair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-key-pairs.html)

Outputs:
```
Sonarqube-Server "ssh -i ~/Window2.pem ubuntu@54.92.66.231"
```

### 3. Config sonarqube server
Copy `<your-sonarqube-server-public-ip>:9000` and open it in your browser

Login with default username `admin` and default password `admin`

Navigate to `Project` > `Manually`

* Project display name : Whatever you want > `Set Up` > `With Github Actions`

Add file `sonar-project.properties` to you repo with content :
```
sonar.projectKey=<your-project-key>
```
Click `Continue`

Click `Generate a token` > `Copy`

Go back to github `Settings` > `Secrets and variables` > `Actions` > `New repository secret` > Enter key and name
```
SONAR_TOKEN <sonarqube-token>
SONAR_HOST_URL <sonarqube-server-url>
```
Click `Continue`

You can use [github-cli](https://cli.github.com/)

First, you need to create github token . [github-token](https://docs.github.com/en/enterprise-server@3.9/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)

Then login with command :
```
gh auth login
```
Then `cd` to your `github-repo`
```
gh secret set SONAR_TOKEN <sonarqube-token>
gh secret set SONAR_HOST_URL <sonarqube-server-url>
```
### 3. Dockerhub config

Navigate to Dockerhub > `Finger print in top right corner` >  `Account Settings` > `Security` > `New Access Token` > Enter name > `Generate` > Copy it

Go back to github `Settings` > `Secrets and variables` > `Actions` > `New repository secret` > Enter key and name

```
DOCKER_USERNAME <your-dockerhub-name>
DOCKER_PASSWORD <your-docker-token>
```
You can use [github-cli](https://cli.github.com/)
```
gh secret set DOCKER_USERNAME <your-dockerhub-name>
gh secret set DOCKER_PASSWORD <your-docker-token>
```
### 4. Install Runner
Navigate to github > `Settings` > `Action` > `Runner` > `New self-host runner`
In runner image we choose `Linux`

Copy the code, ssh to Sonarqube-server and run it

Go back to `github-repo` repo and make a commit to start build
add file `.github/workflow/builf.yml`
```
name: Build

on:
  push:
    branches:
      - main


jobs:
  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
        with:
          fetch-depth: 0
      - name: Build and analyze it with SonarQube
        uses: sonarsource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
      - name: install trivy
        run: |
          #install trivy
          sudo apt-get install wget apt-transport-https gnupg lsb-release -y
          wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
          echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
          sudo apt-get update
          sudo apt-get install trivy -y
          trivy fs .
      - name: Docker build and push
        run: |
          docker build -f Dockerfile . -t haquocdat543/action:latest
          docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}
          docker push haquocdat543/action:latest
        env:
          DOCKER_CLI_ACI: 1
      - name: Image scan
        run: trivy image haquocdat/action:latest > trivyimage.txt
  deploy:    
      needs: build
      runs-on: self-hosted  
      steps:
        - name: Pull the docker image
          run: docker pull haquocdat543/action:latest
        - name: Trivy image scan
          run: trivy image haquocdat543/action:latest
        - name: Run the container vue
          run: docker run -d --name action -p 8080:8080 haquocdat543/action:latest
```

```
git add .
git commit -m "Add sonar-project.properties and workflow"
git push
```

Go back to github > `Action` to see result

Check dockerhub:

Copy `<sonarqube-server-ip>:8080` to access vue app

Copy mail :

### 5. Destroy
```
terraform destroy --auto-approve
```

