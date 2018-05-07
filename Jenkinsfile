properties(
  [
    parameters(
      [
        string(
          defaultValue: 'x86_64,ppc64le',
          description: 'A comma separated list of architectures to run the test on. Valid values include [x86_64, ppc64le, aarch64, s390x].',
          name: 'ARCHES'
        ),
        string(
          defaultValue: 'https://github.com/RedHat-MultiArch-QE/multiarch-ci-libraries',
          description: 'Repo for shared libraries.',
          name: 'LIBRARIES_REPO'
        ),
        string(
          defaultValue: 'v1.0',
          description: 'Git reference to the branch or tag of shared libraries.',
          name: 'LIBRARIES_REF'
        ),
        string(
          defaultValue: '',
          description: 'Repo for tests to run. If left blank, the current repo is assumed (*note* this default will only work for multibranch pipelines).',
          name: 'TEST_REPO'
        ),
        string(
          defaultValue: '',
          description: 'Git reference to the branch or tag of the tests repo.',
          name: 'TEST_REF'
        ),
        string(
          defaultValue: 'tests',
          description: 'Directory containing tests to run. Should at least one of the follow: an ansible-playbooks directory containing one or more test directories each of which having a playbook.yml, a scripts directory containing one or more test directories each of which having a run-test.sh',
          name: 'TEST_DIR'
        ),
        string(
          defaultValue: '',
          description: 'Contains the CI_MESSAGE for a message bus triggered build.',
          name: 'CI_MESSAGE'
        )
      ]
    )
  ]
)

library(
  changelog: false,
  identifier: "multiarch-ci-libraries@${params.LIBRARIES_REF}",
  retriever: modernSCM([$class: 'GitSCMSource',remote: "${params.LIBRARIES_REPO}"])
)

List arches = params.ARCHES.tokenize(',')
def config = TestUtils.getProvisioningConfig(this)
config.runOnSlave = false

TestUtils.runParallelMultiArchTest(
  this,
  arches,
  config,
  { host ->
    /*********************************************************/
    /* TEST BODY                                             */
    /* @param host               Provisioned host details.   */
    /*********************************************************/
    dir('test') {
      stage ('Download Test Files') {
        if (params.TEST_REPO) {
          git url: params.TEST_REPO, branch: params.TEST_REF, changelog: false
        }
        else {
          checkout scm
        }
      }

      stage ('Run Test') {
        if (config.runOnSlave) {
          sh "ansible-playbook -i 'localhost,' -c local ${TEST_DIR}/ansible-playbooks/*/playbook.yml"
          sh "for i in ${TEST_DIR}/scripts/*/test.sh; do bash \$i; done"
        }
        else {
          sh "ansible-playbook -i '${host.inventory}' ${TEST_DIR}/ansible-playbooks/*/playbook.yml"
          sh "for i in ${TEST_DIR}/scripts/*/test.sh; do ssh root@${host.hostName} < \$i; done"
        }
      }

      stage ('Archive Test Output') {
        try {
          archiveArtifacts allowEmptyArchive: true, artifacts: "${params.TEST_DIR}/ansible-playbooks/**/artifacts/*", fingerprint: true
          junit "${params.TEST_DIR}/ansible-playbooks/**/reports/*.xml"
        }
        catch (e) {
          // We don't care if this step fails
        }
        try {
          archiveArtifacts allowEmptyArchive: true, artifacts: "${params.TEST_DIR}/scripts/**/artifacts/*", fingerprint: true
          junit "${params.TEST_DIR}/scripts/**/reports/*.xml"
        }
        catch (e) {
          // We don't care if this step fails
        }
      }
    }

    /*****************************************************************/
    /* END TEST BODY                                                 */
    /* Do not edit beyond this point                                 */
    /*****************************************************************/
  },
  { Exception exception, def host ->
    echo "Exception ${exception} occured on ${host.arch}"
    if (host.arch.equals("x86_64") || host.arch.equals("ppc64le")) {
      currentBuild.result = 'FAILURE'
    }
  }
)
