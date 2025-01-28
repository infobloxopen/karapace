@Library('jenkins.shared.library') _

pipeline {
  agent {
    label 'ubuntu_20_04_label'
  }
  tools {
    go "Go 1.22.4"
  }
  stages {
    stage("Setup") {
      steps {
        prepareBuild()
      }
    }
    stage("Build Image") {
      steps {
        dir("${PROJECT}") {
          sh "make docker-build"
        }
      }
    }
  }
  post {
    success {
      script {
        dir("${PROJECT}") {
          finalizeBuild(
            sh(
              script: 'make image/show',
              returnStdout: true
            )
          )
        }
      }
    }
    cleanup {
      cleanWs()
    }
  }
}
