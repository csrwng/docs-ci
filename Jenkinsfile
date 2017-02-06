node {
  stage("Checkout") {
    checkout scm
  }
  stage("Build Jenkins Base") {
    def sourceUrl = "https://github.com/csrwng/jenkins.git"
    def sourceRef = "v2.32"
    def name = "jenkins-base"
    def params = "-p NAME=${name}"
    params += " -p SOURCE_URL=${sourceUrl}"
    params += " -p SOURCE_REF=${sourceRef}"
    sh "oc new-app -f openshift/jenkins-docker-build-template.yaml ${params}"
    timeout(10) {
      sh "while(true); do if oc get is centos -o jsonpath='{ .status.tags[*].tag }' | grep -q centos7; then break; fi; done"
    }
    sh "oc start-build bc/${name} --follow"
  }
  stage("Build Jenkins with Plugins") {
    def sourceUrl = "https://github.com/csrwng/docs-ci.git"
    def sourceRef = "master"
    def sourceContext = "jenkins"
    def name = "docs-ci-jenkins"
    def params = "-p NAME=${name}"
    params += " -p SOURCE_URL=${sourceUrl}"
    params += " -p SOURCE_REF=${sourceRef}"
    params += " -p SOURCE_CONTEXT=${sourceContext}"
    params += " -p BASE_NAME=jenkins-base"
    params += " -p BASE_TAG=latest"
    params += " -p BASE_NAMESPACE=\"\""
    sh "oc new-app -f openshift/jenkins-s2i-build-template.yaml ${params}"
    sh "oc start-build bc/${name} --follow --from-dir=./jenkins"
  }
  stage("Deploy Jenkins") {
    def params = "-p ENABLE_OAUTH=false"
    params += " -p MEMORY_LIMIT=2Gi"
    params += " -p NAMESPACE=\"\""
    params += " -p JENKINS_IMAGE_STREAM_TAG=docs-ci-jenkins:latest"
    params += " -p JENKINS_SERVICE_NAME=docs-ci-jenkins"
    params += " -p JNLP_SERVICE_NAME=docs-ci-jnlp"
    sh "oc new-app jenkins-persistent ${params}"
    openshiftVerifyDeployment depCfg: "docs-ci-jenkins"
  }
}
