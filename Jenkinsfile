def eng_account = "895523100917"

def project = [:]
project.config       = 'hmpps-env-configs'
project.oracle_patch = 'hmpps-oracle-database-patches'
project.oracle_vars  = 'hmpps-delius-core-oracledb-bootstrap'

def environments = [
  'delius-core-sandpit',
  'delius-core-dev',
  'delius-auto-test',
  'delius-test',
  'delius-stage',
  'delius-mis-dev',
  'delius-mis-test',
  'delius-po-test1',
  'delius-training',
  'delius-training-test',
  'delius-perf',
  'delius-pre-prod',
  'delius-prod'
]

def targets = [
  'delius_primarydb',
  'delius_standbydb1',
  'delius_standbydb2',
  'mis_primarydb',
  'mis_standbydb1',
  'mis_standbydb2', 
  'misboe_primarydb',
  'misboe_standbydb1',
  'misboe_standbydb2', 
  'misdsd_primarydb',
  'misdsd_standbydb1',
  'misdsd_standbydb2',
  'delius_dbs',
  'mis_dbs',
  'misboe_dbs',
  'misdsd_dbs'
]

def prepare_env() {
    sh '''
        docker pull mojdigitalstudio/hmpps-ansible-builder:latest
    '''
}

def run_ansible(environment_name,target_host,patch_id,install_absent_patches) {

    // If the patch_id is set to ALL, unset it so it is not passed as an extra var
    // to ansible.  Ansible will then treat install_patch_id as undefined and process
    // for all configured patches.
    sshagent (credentials: ['hmpps_integration_test-key']) {
        sh """
        #!/usr/env/bin bash
        set +x
        docker run --rm \
        -v `pwd`:/home/tools/data \
        -v ~/.ssh:/home/tools/.ssh \
        -v $SSH_AUTH_SOCK:/ssh-agent \
        -e SSH_AUTH_SOCK=/ssh-agent \
        mojdigitalstudio/hmpps-ansible-builder \
        bash -c \"export ANSIBLE_CONFIG=/home/tools/data/hmpps-env-configs/ansible/ansible.cfg && \
                  export _PATCH_ID=${patch_id} && \
                  export SPECIFIC_PATCH_ID=\\\$(echo \\\${_PATCH_ID} | sed 's/^ALL\\\$//') && \
                  ansible-playbook -u hmpps_integration_test \
                  -i "/home/tools/data/hmpps-env-configs/${environment_name}/ansible" \
                  ./delius-oracle-database-patches/oneoffpatch.yml \
                  --extra-vars "oracle_vars=/home/tools/data/hmpps-delius-core-oracledb-bootstrap/vars/main.yml" \
                  --extra-vars "install_absent_patches=${install_absent_patches}" \
                  --extra-vars "target_host=${target_host}" \
                  --extra-vars "environment_name=${environment_name}" \
                  \\\${SPECIFIC_PATCH_ID:+--extra-vars install_patch_id=\\\${SPECIFIC_PATCH_ID}} \
                  \"
        set -x
        """
    }
}

pipeline {

  agent { label "jenkins_slave" }

  parameters {
    choice(name: 'environment_name', choices: environments, description: 'Select environment')
    choice(name: 'target_host', choices: targets, description: 'Select database')
    string(name: 'patch_id',, description: 'OPTIONAL [ID of Patch to be Installed (Patch should already be available in S3 bucket)].  If not specified then all configured patches will be installed.')
    booleanParam(name: 'install_absent_patches', defaultValue: false, description: 'Install any patches found to be absent according to the configuration for this environment.  By default no patches are installed and instead an error is returned if any are missing.')
  }

  stages {
    stage('Setup') {
      steps {
        dir( project.config ) {
          git url: 'git@github.com:ministryofjustice/' + project.config, branch: env.GIT_BRANCH.split(/\//)[1], credentialsId: 'f44bc5f1-30bd-4ab9-ad61-cc32caf1562a'
        }
        dir( project.d_man_dep ) {
          git url: 'git@github.com:ministryofjustice/' + project.oracle_patch, branch: env.GIT_BRANCH.split(/\//)[1], credentialsId: 'f44bc5f1-30bd-4ab9-ad61-cc32caf1562a'
        }
        dir( project.oracle_vars ) {
          git url: 'git@github.com:ministryofjustice/' + project.oracle_vars, branch: 'master', credentialsId: 'f44bc5f1-30bd-4ab9-ad61-cc32caf1562a'
        }
        prepare_env()
      }
    }

    stage('Install Oracle One-Off Patches') {
      steps {
        run_ansible(environment_name,target_host,patch_id,install_absent_patches)
      }
    }
  }

  post {
    always {
      deleteDir()
      }
    failure {
      slackSend(message: "Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}", color: 'danger')
    }
  }
}
