#!/usr/bin/env groovy

node {
  def project="${env.PROJECT_NAME}"
  stage("Input Parameters") {
    params = input(message: "Enter CI deploy parameters", parameters: [
      string(
        defaultValue: '2Gi',
        description: 'Jenkins docs-ci memory',
        name: 'jenkinsMemory'
      ),
      string(
        defaultValue: 'https://github.com/csrwng/jenkins.git', 
        description: 'Git URL of the base Jenkins image Docker repository to use for CI', 
        name: 'jenkinsBaseSourceUrl'
      ), 
      string(
        defaultValue: 'v2.32', 
        description: 'Git Ref of the base Jenkins image Docker repository to use for CI', 
        name: 'jenkinsBaseSourceRef'
      ),
      string(
        defaultValue: 'https://github.com/csrwng/docs-ci.git', 
        description: 'Git URL of the repository that contains Jenkins customizations for docs-ci', 
        name: 'jenkinsDocsCISourceUrl'
      ),
      string(
        defaultValue: 'master', 
        description: 'Git Ref of the repository that contains Jenkins customizations for docs-ci', 
        name: 'jenkinsDocsCISourceRef'
      ),
      string(
        defaultValue: 'jenkins', 
        description: 'Git Context Directory of the repository that contains Jenkins customizations for docs-ci', 
        name: 'jenkinsDocsCIContext'
      )
    ])
  }
  stage("Checkout") {
    checkout scm
  }
  stage("Build Jenkins Base") {
    def sourceUrl = params.jenkinsBaseSourceUrl
    def sourceRef = params.jenkinsBaseSourceRef
    def name = "jenkins-base"
    def params = "-p NAME=${name}"
    params += " -p SOURCE_URL=${sourceUrl}"
    params += " -p SOURCE_REF=${sourceRef}"
    sh "oc new-app -f openshift/jenkins-docker-build-template.yaml ${params} --dry-run -o yaml -n ${project} | oc apply -n ${project} -f - "
    timeout(10) {
      sh "while(true); do if oc get is -n ${project} centos -o jsonpath='{ .status.tags[*].tag }' | grep -q centos7; then break; fi; done"
    }
    def build; 
    // Start a build
    build = sh(script: "oc start-build bc/${name} -n ${project}", returnStdout: true).trim()
    // Wait for the build to not be in the New or Pending state
    timeout(5) {
      sh "while(true); do if oc get build ${build} -n ${project} -o jsonpath='{ .status.phase }' | egrep -qv 'New|Pending'; then break; fi; done"
    }
    sh "oc logs -n ${project} -f build/${build}"
    // Verify that the build is in the Complete state
    sh "oc get build ${build} -n ${project} -o jsonpath = '{ .status.phase }' | grep -q Complete"
  }
  stage("Build Jenkins with Plugins") {
    def sourceUrl = params.jenkinsDocsCISourceUrl
    def sourceRef = params.jenkinsDocsCISourceRef
    def sourceContext = params.jenkinsDocsCISourceContext
    def name = "docs-ci-jenkins"
    def params = "-p NAME=${name}"
    params += " -p SOURCE_URL=${sourceUrl}"
    params += " -p SOURCE_REF=${sourceRef}"
    params += " -p SOURCE_CONTEXT=${sourceContext}"
    params += " -p BASE_NAME=jenkins-base"
    params += " -p BASE_TAG=latest"
    params += " -p BASE_NAMESPACE=\"\""
    sh "oc new-app -f openshift/jenkins-s2i-build-template.yaml ${params} --dry-run -o yaml -n ${project} | oc apply -n ${project} -f - "
    def build; 
    // Start a build
    build = sh(script: "oc start-build bc/${name} -n ${project}", returnStdout: true).trim()
    // Wait for the build to not be in the New or Pending state
    timeout(5) {
      sh "while(true); do if oc get build ${build} -n ${project} -o jsonpath='{ .status.phase }' | egrep -qv 'New|Pending'; then break; fi; done"
    }
    sh "oc logs -n ${project} -f build/${build}"
    // Verify that the build is in the Complete state
    sh "oc get build ${build} -n ${project} -o jsonpath = '{ .status.phase }' | grep -q Complete"
  }
  stage("Deploy Jenkins") {
    def memory = params.jenkinsMemory
    def params = "-p MEMORY_LIMIT=${memory}"
    params += " -p NAMESPACE=\"\""
    params += " -p JENKINS_IMAGE_STREAM_TAG=docs-ci-jenkins:latest"
    params += " -p JENKINS_SERVICE_NAME=docs-ci-jenkins"
    params += " -p JNLP_SERVICE_NAME=docs-ci-jnlp"
    sh "oc new-app jenkins-persistent ${params} --dry-run -o yaml -n ${project} | oc apply -n ${project} -f - "
    openshiftVerifyDeployment depCfg: "docs-ci-jenkins"
  }
}
