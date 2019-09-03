pipeline {
  agent { node { label 'gradle-ansible-jdk11' } }

  environment {
    APP_NAME = 'akka-cluster-demo'
    ARTIFACT_PASS = credentials('jenkins-artifactory-password')
    ARTIFACTORY_PARAMS = "-PARTIFACT_USER=jenkins-reboot -PARTIFACT_PASS=${ARTIFACT_PASS}"
  }

  stages {
    stage('Init') {
      steps {
        timestamps {
          sh "git clean -xdf"
          script {
            commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
          }
        }
      }
    }

    stage('QA') {
      steps {
        timestamps {
          checkout scm
          script {
            commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
          }
          script {
            vers = sh(returnStdout: true, script: './mvnw help:evaluate -q -Dexpression=project.version -DforceStdout').trim()
            echo "project.version: ${vers}"
          }
          sh './mvnw clean test'
        }
      }
      post {
       always {
            archive "target/**/*"
            junit 'target/surefire-reports/*.xml'
        }
      }
    }

    stage('Jar') {
      steps {
        timestamps {
          sh './mvnw package'
        }
      }
    }

    stage('Install') {
      steps {
        timestamps {
          ansiColor('xterm') {
            ansiblePlaybook(
                installation: 'ansible-playbook',
                inventory: 'install/inventories/bld',
                playbook: 'install/site.yml',
                colorized: true)
          }
        }
      }
    }

    stage('Build Image') {
      steps {
        timestamps {
          sh "mkdir -p s2i-files"
          sh "rm -rf s2i-files/*.jar"
          sh "ls -l target/*-shaded.jar"
          sh "cp target/*-shaded.jar s2i-files"
          script {
              openshift.withCluster() {openshift.startBuild( APP_NAME, '--from-dir=s2i-files/', '--wait')}
          }
        }
      }
    }

    stage('Config Tst') {
      steps {
        ansiColor('xterm') {
          ansiblePlaybook(
              installation: 'ansible-playbook',
              inventory: 'install/inventories/tst',
              playbook: 'install/site.yml',
              colorized: true)
        }
      }
    }

    stage('Deploy Tst') {
      steps {
        script {
           openshift.withCluster() {
              openshift.tag("${APP_NAME}:latest", "${APP_NAME}:tst")
           }
        }
        script {
          sleep 10
          openshift.withCluster() {
            openshift.withCredentials('ansible-token-reboot') {
              openshift.withProject('reboot-tst'){
                def latestDeploymentVersion = openshift.selector('dc', APP_NAME).object().status.latestVersion
                def rc = openshift.selector('rc', "${APP_NAME}-${latestDeploymentVersion}")
                rc.untilEach(1){
                  def rcMap = it.object()
                  return (rcMap.status.replicas.equals(rcMap.status.readyReplicas))
                }
              }
            }
          }
        }
      }
    }
  }
}
