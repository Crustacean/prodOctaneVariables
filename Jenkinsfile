String targetRepositoryUrl =
    params.DIR1_REPOSITORY_URL?.toString()?.trim() ?: 'https://github.com/Crustacean/dashboardMain.git'
String targetBranch = params.DIR1_BRANCH?.toString()?.trim() ?: 'main'
String targetCredentialsId =
    params.DIR1_CREDENTIALS_ID?.toString()?.trim() ?: 'pushToGit'
String transportedConfigurationJson = ''

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
    Map configuration = copiedConfiguration as Map
    List<String> configurationKeys =
        configuration.keySet().collect { Object key -> key.toString().trim() }.sort()
    List<String> invalidKeys =
        configurationKeys.findAll { String key -> !(key ==~ /[A-Z][A-Z0-9_]*/) }
    if (!invalidKeys.isEmpty()) {
      error "Configuration keys must use upper snake case: ${invalidKeys.join(', ')}"
    }
    List<String> nonScalarKeys =
        configurationKeys.findAll { String key ->
          def value = configuration.get(key)
          return value instanceof Map || value instanceof Collection
        }
    if (!nonScalarKeys.isEmpty()) {
      error "Configuration values must be scalar: ${nonScalarKeys.join(', ')}"
    }

    transportedConfigurationJson = writeJSON(json: configuration, returnText: true)
    echo "Copied and validated ${sourceConfiguration} as ${targetConfiguration}."
  }

  stage('Execute dir1 Pipeline') {
    dir('dir1') {
      if (!fileExists('Jenkinsfile')) {
        error 'Target pipeline was not found: dir1/Jenkinsfile'
      }
      String pipelineSourceDirectory = pwd()
      List<String> pipelineEnvironment = [
        "OCTANE_BOOTSTRAP_CONFIGURATION_JSON=${transportedConfigurationJson}",
        "OCTANE_PIPELINE_SOURCE_DIR=${pipelineSourceDirectory}"
      ]
      echo "Executing target Pipeline from ${pipelineSourceDirectory}."
      withEnv(pipelineEnvironment) {
        load 'Jenkinsfile'
      }
    }
  }
}
