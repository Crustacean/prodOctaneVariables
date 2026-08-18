String targetRepositoryUrl =
    params.DIR1_REPOSITORY_URL?.toString()?.trim() ?: 'https://github.com/Crustacean/dashboardMain.git'
String targetBranch = params.DIR1_BRANCH?.toString()?.trim() ?: 'main'
String targetCredentialsId =
    params.DIR1_CREDENTIALS_ID?.toString()?.trim() ?: 'pushToGit'

if (!targetRepositoryUrl) {
  error 'DIR1_REPOSITORY_URL must identify the repository containing Jenkinsfile.'
}

node {
  stage('Checkout Repositories') {
    deleteDir()

    dir('dir2') {
      checkout scm
    }

    dir('dir1') {
      Map<String, String> remote = [url: targetRepositoryUrl]
      if (targetCredentialsId) {
        remote.credentialsId = targetCredentialsId
      }
      checkout([
        $class: 'GitSCM',
        branches: [[name: targetBranch.startsWith('refs/') ? targetBranch : "*/${targetBranch}"]],
        doGenerateSubmoduleConfigurations: false,
        extensions: [
          [$class: 'CleanBeforeCheckout'],
          [$class: 'PruneStaleBranch']
        ],
        userRemoteConfigs: [remote]
      ])
    }
  }

  stage('Copy Configuration') {
    String sourceConfiguration = 'dir2/variables.yaml'
    String targetConfiguration = 'dir1/variables.yaml'
    if (!fileExists(sourceConfiguration)) {
      error "Source configuration was not found: ${sourceConfiguration}"
    }

    sh 'cp -- dir2/variables.yaml dir1/variables.yaml && chmod 600 dir1/variables.yaml'

    if (!fileExists(targetConfiguration)) {
      error "Configuration copy failed: ${targetConfiguration}"
    }
    def copiedConfiguration = readYaml(file: targetConfiguration)
    if (!(copiedConfiguration instanceof Map)) {
      error "Copied configuration must contain a top-level YAML map: ${targetConfiguration}"
    }
    echo "Copied and validated ${sourceConfiguration} as ${targetConfiguration}."
  }

  stage('Execute dir1 Pipeline') {
    dir('dir1') {
      if (!fileExists('Jenkinsfile')) {
        error 'Target pipeline was not found: dir1/Jenkinsfile'
      }
      env.OCTANE_PIPELINE_SOURCE_DIR = pwd()
      echo "Executing target Pipeline from ${env.OCTANE_PIPELINE_SOURCE_DIR}."
      load 'Jenkinsfile'
    }
  }
}
