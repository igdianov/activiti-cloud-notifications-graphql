pipeline {
    agent {
        kubernetes {
              // Change the name of jenkins-maven label to be able to use yaml configuration snippet
              label "maven-dind"
              // Inherit from Jx Maven pod template
              inheritFrom "maven-java11"
              // Add pod configuration to Jenkins builder pod template
              yamlFile "maven-dind.yaml"
        }
    }
    environment {
      ORG                  = "activiti"
      APP_NAME             = "activiti-cloud-notifications-graphql"
      CHARTMUSEUM_CREDS    = credentials("jenkins-x-chartmuseum")
      GITHUB_CHARTS_REPO   = "https://github.com/${ORG}/activiti-cloud-helm-charts.git"
      GITHUB_HELM_REPO_URL = "https://${ORG}.github.io/activiti-cloud-helm-charts/"
      RELEASE_BRANCH       = "master"
    }
    stages {
      stage("CI Build and push snapshot") {
        when {
          branch "PR-*"
        }
        environment {
          PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
          PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
          HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
        }
        steps {
          container("maven") {
            sh "mvn versions:set -DnewVersion=$PREVIEW_VERSION"
            sh "mvn install"
            sh "export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold.yaml"

            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"

            // Let's build chart to check for any errors
            dir("./charts/$APP_NAME") {
              sh "make build"
            }
          }
        }
      }
      stage("Build Release") {
        when {
          branch "${RELEASE_BRANCH}"
        }
        steps {
          container("maven") {
            // ensure we're not on a detached head
            sh "git checkout ${RELEASE_BRANCH}"
            sh "git config --global credential.helper store"

            sh "jx step git credentials"
            // so we can retrieve the version in later steps
            sh "echo \$(jx-release-version) > VERSION"
            sh "mvn versions:set -DnewVersion=\$(cat VERSION)"

            sh "mvn clean install"

            dir ("./charts/$APP_NAME") {
              sh "make tag"
            }
            sh "mvn deploy -DskipTests"

            sh "export VERSION=`cat VERSION` && skaffold build -f skaffold.yaml"

            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"
          }
        }
      }
      stage("Promote to Environments") {
        when {
          branch "${RELEASE_BRANCH}"
        }
        steps {
          container("maven") {
            dir ("./charts/$APP_NAME") {
              sh "jx step changelog --version v\$(cat ../../VERSION)"

              // publish to github
              sh "make github"

              // Update versions
              sh "make updatebot/push-version"

            }
          }
        }
      }
    }  
    
    post {
        always {
            cleanWs()
        }
    }
}
