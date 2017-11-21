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
          sh 'mvn clean install -B'
      }
    }
    stage('Build container') {
      agent any
      steps {
        script {
            pom = readMavenPom file: 'pom.xml'
            TAG = pom.version
            sh "docker build -t petclinic:${TAG} ."
        }
      }
    }
  }
}
