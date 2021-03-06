#!/usr/bin/env groovy

node {
  def project="${env.PROJECT_NAME}"
  stage("Check pre-conditions") {
    token=readFile("/var/run/secrets/kubernetes.io/serviceaccount/token").trim()
    caPath="/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
    // Check that the docs-ci-jenkins service account can create new projects. Otherwise, fail
    try {
      sh "oc policy can-i -q create projectrequests --as system:serviceaccount:${project}:docs-ci-jenkins --certificate-authority=${caPath} --token=${token}"
    } catch(e) {
      echo "Please ensure that the docs-ci-jenkins service account has permission to create new projects"
      echo "You can add this access with the following command: \n"
      echo "--------------------------------------------------------------------------------------------------------"
      echo "oc adm policy add-cluster-role-to-user self-provisioner system:serviceaccount:${project}:docs-ci-jenkins"
      echo "--------------------------------------------------------------------------------------------------------\n"
      echo "After doing that, re-run this pipeline."
      throw e
    }
  }
  stage("Input Parameters") {
    params = input(message: "Enter CI deploy parameters", parameters: [
      string(
        defaultValue: 'cwbotbot',
        description: 'GitHub User',
        name: 'githubUser'
      ),
      password(
        defaultValue: '',
        description: 'GitHub Token',
        name: 'githubToken'
      ),
      string(
        defaultValue: 'docs',
        description: 'Prefix to use when creating new preview projects',
        name: 'projectPrefix'
      ),
      string(
        defaultValue: '2Gi',
        description: 'Jenkins docs-ci memory',
        name: 'jenkinsMemory'
      ),
      string(
        defaultValue: 'https://github.com/cw-paas-dev/openshift-docs',
        description: 'Git project URL for Docs',
        name: 'docsProjectUrl'
      ),
      string(
        defaultValue: 'https://github.com/cw-paas-dev/openshift-docs.git',
        description: 'Git clone URL for Docs',
        name: 'docsRepositoryUrl'
      ),
      string(
        defaultValue: 'cwbotbot',
        description: 'Docs Administrators',
        name: 'docsAdmins'
      ),
      string(
        defaultValue: 'openshift',
        description: 'Docs White-listed Organizations',
        name: 'whitelistOrgs'
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
  stage("Build Jenkins with Plugins") {
    def sourceUrl = params.jenkinsDocsCISourceUrl
    def sourceRef = params.jenkinsDocsCISourceRef
    def sourceContext = params.jenkinsDocsCIContext
    def docsProjectUrl = params.docsProjectUrl
    def docsRepositoryUrl = params.docsRepositoryUrl
    def projectPrefix = params.projectPrefix
    def docsAdmins = params.docsAdmins
    def whitelistOrgs = params.whitelistOrgs

    def name = "docs-ci-jenkins"
    def params = "-p NAME=${name}"
    params += " -p SOURCE_URL=${sourceUrl}"
    params += " -p SOURCE_REF=${sourceRef}"
    params += " -p SOURCE_CONTEXT=${sourceContext}"
    params += " -p BASE_NAME=jenkins"
    params += " -p BASE_TAG=latest"
    params += " -p BASE_NAMESPACE=\"\""
    sh "oc new-app -f openshift/jenkins-image-stream.yaml --dry-run -o yaml -n ${project} | oc apply -n ${project} -f - "
    sh "oc new-app -f openshift/jenkins-s2i-build-template.yaml ${params} --dry-run -o yaml -n ${project} | oc apply -n ${project} -f - "

    // Transform job templates
    def env = [
      "GITHUB_PROJECT_URL=${docsProjectUrl}",
      "GITHUB_REPOSITORY_URL=${docsRepositoryUrl}",
      "PROJECT_ADMINS=${docsAdmins}",
      "WHITELIST_ORGS=${whitelistOrgs}",
      "CI_REPOSITORY_URL=${sourceUrl}",
      "PROJECT_PREFIX=${projectPrefix}"
    ]

    def vars='$GITHUB_PROJECT_URL,$GITHUB_REPOSITORY_URL,$PROJECT_ADMINS,$WHITELIST_ORGS,$CI_REPOSITORY_URL,$PROJECT_PREFIX'

    withEnv(env) {
      sh "cat jenkins/configuration/jobs/docs-pr-test/config.xml.template | envsubst '" + vars + "' > jenkins/configuration/jobs/docs-pr-test/config.xml"
      sh "cat jenkins/configuration/jobs/docs-pr-trigger/config.xml.template | envsubst '" + vars + "' > jenkins/configuration/jobs/docs-pr-trigger/config.xml"
    }

    def build;
    // Start a build
    build = sh(script: "oc start-build bc/${name} -n ${project} --from-dir=${sourceContext} -o name", returnStdout: true).trim()
    // Wait for the build to not be in the New or Pending state
    echo "Waiting for ${build} to start..."
    timeout(5) {
      sh "set +x; while(true); do if oc get ${build} -n ${project} -o jsonpath='{ .status.phase }' | egrep -qv 'New|Pending'; then break; fi; sleep 1; done"
    }
    sh "oc logs -n ${project} -f ${build}"

    // Wait for the build to finish running
    timeout(5) {
      sh "set +x; while(true); do if oc get ${build} -n ${project} -o jsonpath='{ .status.phase }' | egrep -qv 'Running'; then break; fi; sleep 1; done"
    }

    // Verify that the build is in the Complete state
    sh "oc get ${build} -n ${project} -o jsonpath='{ .status.phase }' | grep -q Complete"
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
  stage("Setup Jenkins Credentials") {
    echo "Creating Github Credentials"
    def saToken = sh(script:'oc whoami -t', returnStdout:true).trim()
    def githubUser = params.githubUser
    def githubToken = params.githubToken
    sh "set +x; curl " +
       "-X POST " +
       '-H "Authorization: Bearer ' + saToken + '" ' +
       'http://docs-ci-jenkins/credentials/store/system/domain/_/createCredentials ' +
       "--data-urlencode '" +
          'json={ "":"0", ' +
          '"credentials": { ' +
          '   "scope": "GLOBAL", ' +
          '   "id": "github_userpass", ' +
          '   "username": "' + githubUser + '", ' +
          '   "password": "' + githubToken + '", ' +
          '   "description": "", ' +
          '   "$class": "com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl" } }' + "'"
    sh "set +x; curl " +
       "-X POST " +
       '-H "Authorization: Bearer ' + saToken + '" ' +
       'http://docs-ci-jenkins/credentials/store/system/domain/_/createCredentials ' +
       "--data-urlencode '" +
          'json={ "":"0", ' +
          '"credentials": { ' +
          '   "scope": "GLOBAL", ' +
          '   "id": "github_token", ' +
          '   "secret": "' + githubToken + '", ' +
          '   "description": "", ' +
          '   "$class": "org.jenkinsci.plugins.plaincredentials.impl.StringCredentialsImpl" } }' + "'"
  }
}
