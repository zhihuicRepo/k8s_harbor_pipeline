#!/usr/bin/env groovy
@Library('github.com/chinakevinguo/sharelibrary@master') _
def pipeline = new org.homework.Pipeline()
podTemplate(cloud: 'kubernetes',label: 'mypod',namespace: 'jenkins',serviceAccount: 'jenkins-admin',containers: [
    containerTemplate(name: 'jnlp', image: 'harbor.quark.com/quark/jnlp-slave:alpine', workingDir: '/home/jenkins'),
    containerTemplate(name: 'maven', image: 'harbor.quark.com/quark/maven:3.5.0-8u74', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'docker', image: 'harbor.quark.com/quark/docker:1.12.6', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'kubectl', image: 'harbor.quark.com/quark/k8s-kubectl:v1.8.4', ttyEnabled: true, command: 'cat')
    ],
    volumes : [
        [$class: 'HostPathVolume', mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'],
        [$class: 'PersistentVolumeClaim', mountPath: '/root/.m2', claimName: 'jenkins-m2-pvc', readOnly: false]
        ]) {
        node ('mypod') {

          checkout scm
            def config = readYaml file: 'Jenkinsfile.yaml'
            stage('checkout scm') {
              container('maven') {
                dir(config.git.gitlocal){
                  checkout([$class: 'GitSCM',
                  branches: [[name: config.git.gitbranch]],
                  doGenerateSubmoduleConfigurations: false,
                  extensions: [],
                  submoduleCfg: [],
                  userRemoteConfigs: [[credentialsId: config.git.gitcredentialsid,url: config.git.gitrepo]]])
                  }
              }
            }
            stage('build artifics') {
              container('maven') {
                mvnPackage(config.args.mavenoptions)
              }
            }

            stage('build image') {
              container('docker') {
                withCredentials(
                  [[$class: 'UsernamePasswordMultiBinding',
                  credentialsId: 'harbor',
                  usernameVariable: 'HARBOR_USER',
                  passwordVariable: 'HARBOR_PASSWORD'
                  ]]) {
                    sh "docker login harbor.quark.com -u ${env.HARBOR_USER} -p ${env.HARBOR_PASSWORD}"
                    // sh (script: "docker build -t ${config.images.app} ${config.app.dockerfile}",returnStdout: true)
                    // sh (script: "docker push ${config.images.app}",returnStdout: true)
                    // sh (script: "docker rmi ${config.images.app}",returnStdout: true)
                  }
              }
            }

            stage('deploy to k8s') {
              container('kubectl') {
                sh "kubectl apply -f app-deploy.yaml"
                sh "kubectl rollout status deployment/tomcat -n quark-dev"
                sh "kubectl get pods -n quark-dev -o wide"
                sh "kubectl get rs -n quark-dev"
              }
            }
        }
}
