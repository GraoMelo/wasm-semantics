pipeline {
  agent { dockerfile { reuseNode true } }
  options { ansiColor('xterm') }
  stages {
    stage('Init title') {
      when { changeRequest() }
      steps { script { currentBuild.displayName = "PR ${env.CHANGE_ID}: ${env.CHANGE_TITLE}" } }
    }
    stage('Build and Test') {
      when { changeRequest() }
      stages {
        stage('Dependencies') { steps { sh 'make deps'      } }
        stage('Build')        { steps { sh 'make build -j4' } }
        stage('Test') {
          options { timeout(time: 20, unit: 'MINUTES') }
          parallel {
            stage('Simple')            { steps { sh 'make TEST_CONCRETE_BACKEND=llvm test-simple -j4'                } }
            stage('Conformance Parse') { steps { sh 'make test-conformance-parse -j2'                                } }
            stage('Conformance')       { steps { sh 'make TEST_CONCRETE_BACKEND=llvm test-conformance-supported -j4' } }
            stage('Proofs')            { steps { sh 'make test-prove -j2'                                            } }
          }
        }
      }
    }
    stage('Master Release') {
      when { branch 'master' }
      steps {
        build job: 'rv-devops/master', propagate: false, wait: false                                                   \
            , parameters: [ booleanParam(name: 'UPDATE_DEPS_SUBMODULE', value: true)                                   \
                          , string(name: 'PR_REVIEWER', value: 'ehildenb')                                             \
                          , string(name: 'UPDATE_DEPS_REPOSITORY', value: 'runtimeverification/polkadot-verification') \
                          , string(name: 'UPDATE_DEPS_SUBMODULE_DIR', value: 'deps/wasm-semantics')                    \
                          ]
        build job: 'rv-devops/master', propagate: false, wait: false                                    \
            , parameters: [ booleanParam(name: 'UPDATE_DEPS_SUBMODULE', value: true)                    \
                          , string(name: 'PR_REVIEWER', value: 'hjorthjort')                            \
                          , string(name: 'UPDATE_DEPS_REPOSITORY', value: 'kframework/ewasm-semantics') \
                          , string(name: 'UPDATE_DEPS_SUBMODULE_DIR', value: 'deps/wasm-semantics')     \
                          ]
      }
    }
  }
}
