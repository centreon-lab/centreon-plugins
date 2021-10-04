/*
** Variables.
*/

def buildBranch = env.BRANCH_NAME
if (env.CHANGE_BRANCH) {
  buildBranch = env.CHANGE_BRANCH
}

/*
** Functions
*/

def checkoutCentreonBuild(buildBranch) {
  def getCentreonBuildGitConfiguration = { branchName -> [
    $class: 'GitSCM',
    branches: [[name: "refs/heads/${branchName}"]],
    doGenerateSubmoduleConfigurations: false,
    userRemoteConfigs: [[
      $class: 'UserRemoteConfig',
      url: "ssh://git@github.com/centreon/centreon-build.git"
    ]]
  ]}

  dir('centreon-build') {
    try {
      checkout(getCentreonBuildGitConfiguration(buildBranch))
    } catch(e) {
      echo "branch '${buildBranch}' does not exist in centreon-build, then fallback to master"
      checkout(getCentreonBuildGitConfiguration('master'))
    }
  }
}

stage('Deliver sources') {
  node {
    checkoutCentreonBuild(buildBranch)
    dir('centreon-plugins') {
      checkout scm
    }
    sh './centreon-build/jobs/plugins/plugins-source.sh'
    source = readProperties file: 'source.properties'
    env.VERSION = "${source.VERSION}"
    env.RELEASE = "${source.RELEASE}"
    // Run sonarQube analysis
    withSonarQubeEnv('SonarQubeDev') {
      sh './centreon-build/jobs/plugins/plugins-analysis.sh'
    }
    def qualityGate = waitForQualityGate()
    if (qualityGate.status != 'OK') {
      currentBuild.result = 'FAIL'
    }
  }
}

stage('RPMs Packaging') {
  node {
    checkoutCentreonBuild(buildBranch)
    sh './centreon-build/jobs/plugins/plugins-package.sh'
    archiveArtifacts artifacts: 'rpms-centos7.tar.gz'
    archiveArtifacts artifacts: 'rpms-centos8.tar.gz'
    stash name: "rpms-centos7", includes: 'output-centos7/noarch/*.rpm'
    stash name: "rpms-centos8", includes: 'output-centos8/noarch/*.rpm'
    sh 'rm -rf output-centos7 output-centos8'      
  }
}
if ((currentBuild.result ?: 'SUCCESS') != 'SUCCESS') {
  error('Package stage failure.');
}

if ((env.BRANCH_NAME == 'master')) {
  stage('RPMs delivery to unstable') {
    node {
      checkoutCentreonBuild(buildBranch)
      unstash 'rpms-centos7'
      unstash 'rpms-centos8'
      sh './centreon-build/jobs/plugins/plugins-delivery.sh'
    }
  }
  if ((currentBuild.result ?: 'SUCCESS') != 'SUCCESS') {
    error('Delivery stage failure.');
  }
}
