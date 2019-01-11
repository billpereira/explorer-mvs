#!groovy

/**
 * This program and the accompanying materials are made available under the terms of the
 * Eclipse Public License v2.0 which accompanies this distribution, and is available at
 * https://www.eclipse.org/legal/epl-v20.html
 *
 * SPDX-License-Identifier: EPL-2.0
 *
 * Copyright IBM Corporation 2018
 */



@Library('zoe-jenkins-library') _

def isPullRequest = env.BRANCH_NAME.startsWith('PR-')
def isMasterBranch = env.BRANCH_NAME == 'master'

def opts = []
// keep last 20 builds for regular branches, no keep for pull requests
opts.push(buildDiscarder(logRotator(numToKeepStr: (isPullRequest ? '' : '20'))))
// disable concurrent build
opts.push(disableConcurrentBuilds())

// define custom build parameters
def customParameters = []
customParameters.push(credentials(
  name: 'NPM_CREDENTIALS_ID',
  description: 'npm auth token',
  credentialType: 'org.jenkinsci.plugins.plaincredentials.impl.StringCredentialsImpl',
  defaultValue: 'giza-jenkins-basicAuth',
  required: true
))
customParameters.push(string(
  name: 'NPM_USER_EMAIL',
  description: 'npm user email',
  defaultValue: 'giza-jenkins@gmail.com',
  trim: true
))
customParameters.push(booleanParam(
  name: 'NPM_RELEASE',
  description: 'Publish a release or snapshot version. By default, this task will create snapshot. Check this to publish a release version.',
  defaultValue: false
))
customParameters.push(credentials(
  name: 'PAX_SERVER_CREDENTIALS_ID',
  description: 'The server credential used to create PAX file',
  credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl',
  defaultValue: 'TestAdminzOSaaS2',
  required: true
))
customParameters.push(string(
  name: 'PAX_SERVER_IP',
  description: 'The server IP used to create PAX file',
  defaultValue: '172.30.0.1',
  trim: true
))
customParameters.push(string(
  name: 'ARTIFACTORY_SERVER',
  description: 'Artifactory server, should be pre-defined in Jenkins configuration',
  defaultValue: 'gizaArtifactory',
  trim: true
))
opts.push(parameters(customParameters))

// set build properties
properties(opts)

node ('jenkins-slave') {
  currentBuild.result = 'SUCCESS'
  def packageName
  def packageVersion
  def versionId

  try {

    stage('checkout'){
      // checkout source code
      checkout scm

      // check if it's pull request
      echo "Current branch is ${env.BRANCH_NAME}"
      if (isPullRequest) {
        echo "This is a pull request"
      }

      // get package information
      packageName = sh(script: "node -e \"console.log(require('./package.json').name)\"", returnStdout: true).trim()
      packageVersion = sh(script: "node -e \"console.log(require('./package.json').version)\"", returnStdout: true).trim()
      if (params.NPM_RELEASE) {
        versionId = packageVersion
      } else {
        def buildIdentifier = getBuildIdentifier('%Y%m%d%H%M%S', 'master', false)
        versionId = "${packageVersion}-snapshot.${buildIdentifier}"
      }
      echo "Building ${packageName} v${packageVersion}..."
    }

    stage('prepare') {
      // show node/npm version
      sh 'node -v'
      sh 'npm -v'

      ansiColor('xterm') {
        // login to private npm registry
        def npmRegistry = 'https://gizaartifactory.jfrog.io/gizaartifactory/api/npm/npm-release/'
        npmLogin(npmRegistry, params.NPM_CREDENTIALS_ID, params.NPM_USER_EMAIL)

        sh 'npm ci'
      }
    }

    stage('test') {
      ansiColor('xterm') {
        sh 'npm run lint'
        try {
          sh 'npm test'
        } catch (err) {
          error "Test failed: $err"
        } finally {
          // publish test reports
          junit 'target/report.xml'
          cobertura coberturaReportFile: 'coverage/cobertura-coverage.xml',
            sourceEncoding: 'ASCII',
            autoUpdateHealth: false,
            autoUpdateStability: false,
            onlyStable: false,
            failUnhealthy: false,
            failUnstable: false,
            zoomCoverageChart: false,
            conditionalCoverageTargets: '70, 0, 0',
            lineCoverageTargets: '80, 0, 0',
            methodCoverageTargets: '80, 0, 0',
            maxNumberOfBuilds: 0
          publishHTML([
            allowMissing: false,
            alwaysLinkToLastBuild: false,
            keepAll: false,
            reportDir: 'coverage/lcov-report',
            reportFiles: 'index.html',
            reportName: 'Coverage HTML Report',
            reportTitles: ''
          ])
        }
      }
    }

    stage('SonarQube analysis') {
      def scannerHome = tool 'sonar-scanner-3.2.0';
      withSonarQubeEnv('sonar-default-server') {
        sh "${scannerHome}/bin/sonar-scanner"
      }
    }

    stage('build') {
      ansiColor('xterm') {
        sh 'npm run prod'
      }
    }

    stage('publish') {
      timeout(time: 30, unit: 'MINUTES') {
        echo "prepare pax workspace..."
        sh "scripts/prepare-pax-workspace.sh"

        echo "creating pax file from workspace..."
        createPax("${packageName}-packaging", "${packageName}-${versionId}.pax",
                  params.PAX_SERVER_IP, params.PAX_SERVER_CREDENTIALS_ID,
                  './pax-workspace', '/zaas1/buildWorkspace', '-x os390')

        echo 'publishing pax file to artifactory...'
        def releaseIdentifier = getReleaseIdentifier()
        def server = Artifactory.server params.ARTIFACTORY_SERVER
        def uploadSpec
        if (params.NPM_RELEASE) {
          uploadSpec = readFile "artifactory-upload-spec.release.json.template"
          uploadSpec = uploadSpec.replaceAll(/\{ARTIFACTORY_VERSION\}/, packageVersion)
          uploadSpec = uploadSpec.replaceAll(/\{RELEASE_IDENTIFIER\}/, releaseIdentifier)
        } else {
          uploadSpec = readFile "artifactory-upload-spec.snapshot.json.template"
          uploadSpec = uploadSpec.replaceAll(/\{ARTIFACTORY_VERSION\}/, packageVersion)
          uploadSpec = uploadSpec.replaceAll(/\{RELEASE_IDENTIFIER\}/, releaseIdentifier)
        }
        def buildInfo = Artifactory.newBuildInfo()
        server.upload spec: uploadSpec, buildInfo: buildInfo
        server.publishBuildInfo buildInfo

        // TODO: tag the branch once we release
      }
    }

    stage('done') {
      // send out notification
      emailext body: "Job \"${env.JOB_NAME}\" build #${env.BUILD_NUMBER} success.\n\nCheck detail: ${env.BUILD_URL}" ,
          subject: "[Jenkins] Job \"${env.JOB_NAME}\" build #${env.BUILD_NUMBER} success",
          recipientProviders: [
            [$class: 'RequesterRecipientProvider'],
            [$class: 'CulpritsRecipientProvider'],
            [$class: 'DevelopersRecipientProvider'],
            [$class: 'UpstreamComitterRecipientProvider']
          ]
    }

  } catch (err) {
    currentBuild.result = 'FAILURE'

    // catch all failures to send out notification
    emailext body: "Job \"${env.JOB_NAME}\" build #${env.BUILD_NUMBER} failed.\n\nError: ${err}\n\nCheck detail: ${env.BUILD_URL}" ,
        subject: "[Jenkins] Job \"${env.JOB_NAME}\" build #${env.BUILD_NUMBER} failed",
        recipientProviders: [
          [$class: 'RequesterRecipientProvider'],
          [$class: 'CulpritsRecipientProvider'],
          [$class: 'DevelopersRecipientProvider'],
          [$class: 'UpstreamComitterRecipientProvider']
        ]

    throw err
  }
}
