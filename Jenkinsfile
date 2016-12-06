#!groovy

properties([
  [$class: 'GithubProjectProperty', displayName: 'Simple Application', projectUrlStr: 'https://github.com/alecharp/simple-app/'],
  buildDiscarder(logRotator(artifactNumToKeepStr: '5', daysToKeepStr: '15'))
])

node {
  stage('Checkout') {
    checkout scm
    commit = sh(returnStdout:true, script:'git rev-parse --short HEAD').trim()
    currentBuild.description = "#${commit}"
  }

  stage('Build') {
    mvn 'clean package -Dmaven.test.skip=true'
    archiveArtifacts 'target/*.jar'
    stash name: 'docker', includes: 'src/main/docker/Dockerfile, target/*.jar'
  }
}

stage('Tests') {
  parallel 'Unit tests': {
    node {
      checkout scm
      mvn 'clean test'
      junit 'target/surefire-reports/*.xml'
    }
  }, 'Integration tests': {
    node {
      checkout scm
      mvn 'clean test-compile failsafe:integration-test'
      junit 'target/failsafe-reports/*.xml'
    }
  }
}

stage('Build Docker img') {
  node {
    unstash 'docker'
    image = docker.build("alecharp/simple-app:${commit}", "-f src/main/docker/Dockerfile .")
  }
}

stage('Validate Docker container') {
  node {
    container = image.run('-P')
    ip = container.port(8080)
  }

  try {
    input message: "See http://${ip}. Is it ok?", ok: 'Publish'
  } finally {
    node { container.stop() }
  }
}

stage('Publish Docker img') {
  node {
    docker.withRegistry('http://localhost:5000') {
      image.push "${commit}"
      milestone label: 'docker-image-latest'
      image.push "latest"
    }
  }
}

def mvn(def goals) {
  withMaven() {
    sh "mvn ${goals}"
  }
}
