Blue ocean plugin for better UI.


Setup agents:

Conencts to 
Jenkins > Configure clouds > Install plugins- AWS Cloud or Docker


Jenkins Master will run on EC2. 

Advantages:
We can setup docker based agents to run each pipeline in parallel and isolated container. 


In Jenkins, when you use a Docker Agent to run a pipeline job, the Docker container is **automatically cleaned
up** after the pipeline finishes executing.

Here's what happens:

1. The pipeline job is triggered and executes inside the Docker container.
2. When the pipeline finishes, the Docker container is automatically stopped and removed by Jenkins.
3. The Docker image used to create the container is not modified or persisted in any way.

This ensures that:

* No unnecessary resources are consumed on your EC2 node.
* No images are left behind, which can help maintain a clean and organized system.
* You don't need to worry about manually cleaning up containers after pipeline execution.


Sonarqube Setup-

1. Install sonarqube on node. 
2. **Create a custom quality gate**: In SonarQube, go to Settings > Quality Gates and create a new quality gate
with the following conditions:
        * Metric: `Security Bugs`
        * Operator: `<=`
        * Value: `0`
3. **Configure the quality gate in your pipeline stage**

Add this to pipeline stage-

pipeline {
    stages {

        stage('Build Docker Image') {
            # dockerfile has build instructions
            steps {
                // Checkout the dev branch
                git 'https://github.com/your-repo.git', branch: 'dev'

                // Build the Docker image from the Dockerfile in the dev branch
                sh 'docker build -t my-app:latest .'
            }
        }

        stage('Run Tests') {
             steps {
                 // Run pytest with coverage inside the built container
                 docker.image('my-app:latest').run(
                     command: "pytest --cov=my_app tests/"
                 )
                 def coverage = sh(script: "docker exec -it my-app:latest pycov -m", returnStdout: true).trim()
                 if (coverage.toFloat() < 0.8) {
                     error("Pytest Coverage is lower than 80%")
                 }
             }
         }
        stage('SonarQube Analysis') {
            steps {
                // ... other steps ...
                sonar(
                    server: 'http://sonarqube.com:9000',
                    projectKey: 'your-project-key',
                    projectName: 'your-project-name',
                    branch: 'dev' // specify the branch to analyze
                )
                sonarqubeQualityGate(
                    gate: 'Security Bugs = 0', // name of the quality gate created in step 1
                    failBuildIfGateViolated: true // fails the build if the quality gate is violated
                )
            }
        }
    }
}
```

Jenkins file- 



Have the pipeline setup to poll GIT SCM for a specific branch eg- dev and trigger a build using jenkinsfile-




Store jenkins file in a repository and load it based on project. For example batch pipeline jenkings file below. Have this jenkings file in code branch-
``` Groovy
node {
    stage('Load Jenkinsfile'){
        checkout([ $class: 'GitSCM', branches: [[name: "dev"]], doGenerateSubmoduleConfigurations: false, 
                   extensions: [[ $class: 'RelativeTargetDirectory', relativeTargetDir: 'Jenkinsfile']],
                   submoduleCfg: [],
                   userRemoteConfigs: [[ credentialsId: 'jenkins-user',
                                         url: 'https://git-codecommit.us-west-2.amazonaws.com/v1/repos/jenkins.git']]
                 ])
        jenkinsfile = load 'Jenkinsfile/batch-vi.pipeline'
    }
}

1. This will make a relativedirectory- JenkinsFile and will clone the repo in it. and then it will load the .pipeline from that directory

```
Above the pipeline file batch-vi.pipeline which will have all pipeline stages-
```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'make build'
            }
        }

        stage('Test') {
            steps {
                sh 'make test'
            }
        }

        stage('Deploy') {
            steps {
                sh 'make deploy'
            }
        }
    }
}
```