@Library('jenkins.shared.library') _

pipeline {
  agent {
    label 'ubuntu_20_04_label'
  }
  tools {
    go "Go 1.22.4"
  }
  options {
    checkoutToSubdirectory('src/github.com/infobloxopen/karapace')
  }
  environment {
    PROJECT     = "src/github.com/infobloxopen/karapace"
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
