#!/bin/env groovy

pipeline {
  agent none
  stages {
    stage('Build') {
      agent {
        docker {
          image 'maven:3.5.0'
        }
      }
      steps {
        configFileProvider([configFile(fileId: 'nexus', variable: 'MAVEN_SETTINGS')]) {
          sh 'mvn -s $MAVEN_SETTINGS clean deploy -DskipTests=true -B'
        }
      }
    }
    stage('Build container') {
      agent any
      steps {
        script {
          if ( env.BRANCH_NAME == 'master' ) {
            pom = readMavenPom file: 'pom.xml'
            TAG = pom.version
          } else {
            TAG = env.BRANCH_NAME
          }
          sh "docker build -t ${env.IMAGE}:${TAG} ."
        }
      }
    }

    stage('Run local container') {
      agent any
      steps {
        sh 'docker rm -f petclinic-tomcat-temp || true'
        sh "docker run -d --network=${LDOP_NETWORK_NAME} --name petclinic-tomcat-temp ${env.IMAGE}:${TAG}"
      }
    }

    stage('Smoke-Test & OWASP Security Scan') {
      agent {
        docker {
          image 'maven:3.5.0'
          args '--network=${LDOP_NETWORK_NAME}'
        }
      }
      steps {
        sh "cd regression-suite && mvn clean -B test -DPETCLINIC_URL=http://petclinic-tomcat-temp:8080/petclinic/"
      }
    }
    stage('Stop local container') {
      agent any
      steps {
        sh 'docker rm -f petclinic-tomcat-temp || true'
      }
    }

    stage('Push to dockerhub') {
      agent any
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'dockerPassword', usernameVariable: 'dockerUsername')]){
          script {
            sh "docker login -u ${env.dockerUsername} -p ${env.dockerPassword}"
            sh "docker push ${env.IMAGE}:${TAG}"
          }
        }
      }
    }

    stage('Deploy to dev') {
      when {
        branch 'master'
      }
      agent any
      steps {
        script {
          deployToEnvironment("ec2-user", "dev.petclinic.liatr.io", "petclinic-deploy-key", env.IMAGE, TAG, "spring-petclinic", "dev.petclinic.liatr.io")
        }
      }
    }

    stage('Smoke test dev') {
      when {
        branch 'master'
      }
      agent {
        docker {
          image 'maven:3.5.0'
          args '--network=${LDOP_NETWORK_NAME}'
        }
      }
      steps {
        sh "cd regression-suite && mvn clean -B test -DPETCLINIC_URL=https://dev.petclinic.liatr.io/petclinic"
        echo "Should be accessible at https://dev.petclinic.liatr.io/petclinic"
      }
    }

    stage('Deploy to qa') {
      when {
        branch 'master'
      }
      agent any
      steps {
        deployToEnvironment("ec2-user", "qa.petclinic.liatr.io", "petclinic-deploy-key", env.IMAGE, TAG, "spring-petclinic", "qa.petclinic.liatr.io")
      }
    }

    stage('Smoke test qa') {
      when {
        branch 'master'
      }
      agent {
        docker {
          image 'maven:3.5.0'
          args '--network=${LDOP_NETWORK_NAME}'
        }
      }
      steps {
        sh "cd regression-suite && mvn clean -B test -DPETCLINIC_URL=https://qa.petclinic.liatr.io/petclinic"
        echo "Should be accessible at https://qa.petclinic.liatr.io/petclinic"
        input 'Deploy to Prod?'
      }
    }

    stage('Blue/Green Prod Deploy') {
      when {
        branch 'master'
      }
      agent {
        dockerfile {
          filename "blue-green/Dockerfile"
        }
      }
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'aws', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
          file(credentialsId: 'petclinic-deploy-key', variable: 'DEPLOY_KEY_PATH')
        ]) {
          script {
            sh "TAG=${TAG} blue-green/blue-green deploy"
          }
        }
      }
    }

    stage('Blue/Green Prod Regression Test') {
      when {
        branch 'master'
      }
      agent {
        dockerfile {
          filename "blue-green/Dockerfile"
        }
      }
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'aws', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
          file(credentialsId: 'petclinic-deploy-key', variable: 'DEPLOY_KEY_PATH')
        ]) {
          script {
            sh "TAG=${TAG} blue-green/blue-green test"
          }
        }
      }
    }

    stage('Blue/Green Prod Toggle Load Balancer') {
      when {
        branch 'master'
      }
      agent {
        dockerfile {
          filename "blue-green/Dockerfile"
        }
      }
      steps {
        input "Toggle Prod Load Balancer?"
        withCredentials([
          usernamePassword(credentialsId: 'aws', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
          file(credentialsId: 'petclinic-deploy-key', variable: 'DEPLOY_KEY_PATH')
        ]) {
          script {
            sh "TAG=${TAG} blue-green/blue-green toggle"
          }
        }
      }
    }
  }
}
