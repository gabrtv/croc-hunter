#!/usr/bin/groovy

podTemplate(label: 'jenkins-pipeline', containers: [
    containerTemplate(name: 'jnlp', image: 'jenkinsci/jnlp-slave:2.62', args: '${computer.jnlpmac} ${computer.name}', workingDir: '/home/jenkins', resourceRequestCpu: '200m', resourceLimitCpu: '200m', resourceRequestMemory: '256Mi', resourceLimitMemory: '256Mi'),
    containerTemplate(name: 'docker', image: 'docker:1.12.6', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'golang', image: 'golang:1.8.3', command: 'cat', ttyEnabled: true),
    //containerTemplate(name: 'helm', image: 'gabrtv.azurecr.io/gabrtv/k8s-helm:v2.6.1', command: 'cat', ttyEnabled: true, envVars: [envVar(key: 'HELM_HOST', value: 'tiller-deploy.kube-system:44134')]),
    containerTemplate(name: 'helm', image: 'gabrtv.azurecr.io/gabrtv/k8s-helm:v2.6.1', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.4.8', command: 'cat', ttyEnabled: true)
  ],
  volumes:[
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
  ]){
  node ('jenkins-pipeline') {

      git_result = checkout scm

      def imageTag = "git-" + git_result["GIT_COMMIT"].take(8)
      def kubeconfigCreds = "kubeconfig"
      def dockerRegistry = "gabrtv.azurecr.io"
      def dockerCreds = "gabrtv.azurecr.io"
      def dockerImage = "gabrtv.azurecr.io/gabrtv/croc-hunter:${imageTag}"
      def appName = "croc-hunter"
      def chartDir = "chart/"
      def regions = ["eastus", "southcentralus", "westeurope"]

      stage('Build') {
          container('docker') {
            withCredentials([[$class : 'UsernamePasswordMultiBinding', 
                              credentialsId: dockerCreds,
                              usernameVariable: 'USERNAME', 
                              passwordVariable: 'PASSWORD']]) {
              sh "docker login -u ${USERNAME} -p ${PASSWORD} ${dockerRegistry}"
              sh "docker build --build-arg VCS_REF=${env.GIT_SHA} --build-arg BUILD_DATE=`date -u +'%Y-%m-%dT%H:%M:%SZ'` -t ${dockerImage} ."
              sh "docker push ${dockerImage}"
            }
          }
      }
      stage('Test') {
          container('helm') {
              println "running tests"
          }
      }
      stage('Deploy') {
          container('helm') {
              withCredentials([[$class: 'FileBinding', credentialsId: 'kubeconfig', variable: 'KUBECONFIG_PATH']]) {
                for (region in regions) {
                    sh "KUBECONFIG=${KUBECONFIG_PATH} kubectl config use-context ${region}"
                    sh "KUBECONFIG=${KUBECONFIG_PATH} kubectl get nodes"
                    println "checking client/server version"
                    sh "KUBECONFIG=${KUBECONFIG_PATH} helm version"
                    println "install or upgrade helm chart"
                    sh "KUBECONFIG=${KUBECONFIG_PATH} helm upgrade --install ${appName} ${chartDir} --set imageTag=${imageTag},region=${region},ingress.appName=${appName},ingress.base=gabrtv.io --namespace=${appName}"
                    echo "application ${appName} successfully deployed. Use \"helm status ${appName}\" to check"
                }
            }

          }
      }
    }
}
