#!groovy

def releasedVersion

node('master') {
  def dockerTool = tool name: 'docker', type: 'org.jenkinsci.plugins.docker.commons.tools.DockerTool'
  withEnv(["DOCKER=${dockerTool}/bin"]) {
    stage('Prepare') {
        deleteDir()
        parallel Checkout: {
            checkout scm
        }, 'Run Zalenium': {
            dockerCmd '''run -d --name zalenium -p 4444:4444 \
            -v /var/run/docker.sock:/var/run/docker.sock \
            --network="host" \
            --privileged dosel/zalenium:3.4.0a start --videoRecordingEnabled false --chromeContainers 1 --firefoxContainers 0'''
        }
    }

    stage('Build') {
        withMaven(maven: 'Maven 3') {
            dir('app') {
                sh 'mvn clean package'
                dockerCmd 'build --tag abhaya-docker-local.jfrog.io/sparktodo:SNAPSHOT .'
            }
        }
    }
    stage('Push Snapshot to JFrog Artifactory'){
      def server = Artifactory.server('abhaya-docker-artifactory')
      def rtDocker = Artifactory.docker server: server
      def buildInfo = rtDocker.push 'https://abhaya.jfrog.io/abhaya/docker-local/sparktodo:SNAPSHOT', 'docker-local'
 
      // Publish the build-info to Artifactory:
      server.publishBuildInfo buildInfo
      
      /*docker.withRegistry('https://abhaya.jfrog.io/abhaya', 'abhaya-jfrog-creds'){
        dockerCmd 'push abhaya-docker-local.jfrog.io/sparktodo'
      }
     // def server = Artifactory.newServer url: 'https://abhaya.jfrog.io/abhaya/docker-local/', credentialsId: 'abhaya-jfrog-creds'
      //dockerCmd 'push abhaya-docker-local.jfrog.io/sparktodo'
    }

    stage('Deploy') {
        stage('Deploy') {
            dir('app') {
                dockerCmd 'run -d -p 9999:9999 --name "snapshot" --network="host" automatingguy/sparktodo:SNAPSHOT'
            }
        }
    }

    stage('Tests') {
        try {
            dir('tests/rest-assured') {
                sh 'chmod a+rwx gradlew'
                sh './gradlew clean test'
            }
        } finally {
            junit testResults: 'tests/rest-assured/build/*.xml', allowEmptyResults: true
            archiveArtifacts 'tests/rest-assured/build/**'
        }

        dockerCmd 'rm -f snapshot'
        dockerCmd 'run -d -p 9999:9999 --name "snapshot" --network="host" automatingguy/sparktodo:SNAPSHOT'

        try {
            withMaven(maven: 'Maven 3') {
                dir('tests/bobcat') {
                    sh 'mvn clean test -Dmaven.test.failure.ignore=true'
                }
            }
        } finally {
            junit testResults: 'tests/bobcat/target/*.xml', allowEmptyResults: true
            archiveArtifacts 'tests/bobcat/target/**'
        }

        dockerCmd 'rm -f snapshot'
        dockerCmd 'stop zalenium'
        dockerCmd 'rm zalenium'
    }

    stage('Release') {
        withMaven(maven: 'Maven 3') {
            dir('app') {
                releasedVersion = getReleasedVersion()
                withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'password', usernameVariable: 'username')]) {
                    sh "git config user.email test@automatingguy.com && git config user.name Jenkins"
                    sh "mvn release:prepare release:perform -Dusername=${username} -Dpassword=${password}"
                }
                dockerCmd "build --tag automatingguy/sparktodo:${releasedVersion} ."
            }
        }
    }

    stage('Deploy @ Prod') {
        dockerCmd "run -d -p 9999:9999 --name 'production' automatingguy/sparktodo:${releasedVersion}"
    }
  }
}

def dockerCmd(args) {
    sh "sudo ${DOCKER}/docker ${args}"
}

def getReleasedVersion() {
    return (readFile('pom.xml') =~ '<version>(.+)-SNAPSHOT</version>')[0][1]
}
