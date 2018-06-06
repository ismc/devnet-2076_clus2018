pipeline {
    agent any
    options {
      disableConcurrentBuilds()
      skipDefaultCheckout()
      ansiColor('xterm')
      lock resource: 'jenkins-wan-testbed'
    }
    environment {
      ANSIBLE_INVENTORY_DIR = "${env.WORKSPACE}/inventory"
      GIT_COMMITTER_NAME = 'scarter-jenkins'
      GIT_COMMITTER_EMAIL = 'jenkins@ismc.io'
    }
    triggers {
      cron('H 2 * * 1-5')
    }
    stages {
        stage('Checkout Code') {
            properties([[$class: 'HudsonNotificationProperty', endpoints: [[buildNotes: 'Starting Build', urlInfo: [urlOrId: ' http://cisco-spark-integration-management-ext.cloudhub.io/api/hooks/8fde6043-69b6-11e8-bf37-06c25f4e7996', urlType: 'PUBLIC']]]]])
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: scm.branches,
                    doGenerateSubmoduleConfigurations: false,
                    extensions: scm.extensions + [[$class: 'SubmoduleOption',
                        parentCredentials: true,
                        disableSubmodules: false,
                        recursiveSubmodules: true,
                        reference: '',
                        trackingSubmodules: false]],
                    /* userRemoteConfigs: scm.userRemoteConfigs */
                    userRemoteConfigs: [[credentialsId: 'scarter-jenkins_key', url: 'git@github.com:ismc/devnet-2076_clus2018.git']]
                ])
                directory ('test') {
                    checkout([$class: 'GitSCM',
                        branches: [[name: '*/master']],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'test']],
                        submoduleCfg: [],
                        userRemoteConfigs: [[credentialsId: 'scarter-jenkins_key', url: 'git@github.com:ismc/inventory-test.git']]
                    ])
                }
            }
        }
/*
        stage('Destroy Testbed') {
            steps {
                script {
                    try {
                        echo 'Destroying Cloud...'
                        retry(3) {
                            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'Ansible (scarter)']]) {
                                ansiblePlaybook colorized: true, playbook: 'destroy-testbed.yml'
                            }
                        }
                    } catch (e) {
                        echo 'Previous testbed not found!'
                    }
                }
            }
        }
*/
        stage('Build Testbed') {
            steps {
                echo 'Building Cloud...'
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'Ansible (scarter)']]) {
                  ansiblePlaybook colorized: true, playbook: 'build-testbed.yml'
                }
                script {
                    try {
                        echo 'Updating Inventory...'
                        dir('inventory') {
                            sshagent (credentials: ['scarter-jenkins_key']) {
                                sh 'git add *'
                                sh 'git commit -am "Updated inventory on re-build"'
                                sh 'git push origin HEAD:master --force'
                            }
                        }
                    } catch (e) {
                            echo 'Unable to commit inventory'
                    }
                }
            }
        }
        stage('Run Tests') {
            steps {
                echo 'Running network-system.yml...'
                ansiblePlaybook credentialsId: 'scarter-jenkins_key', colorized: true, disableHostKeyChecking: true, inventory: "${env.ANSIBLE_INVENTORY_DIR}/test/wan-testbed.yml", playbook: 'network-system.yml'

                echo 'Running network-checkpoint.yml...'
                ansiblePlaybook credentialsId: 'scarter-jenkins_key', colorized: true, disableHostKeyChecking: true, inventory: "${env.ANSIBLE_INVENTORY_DIR}/test/wan-testbed.yml", playbook: 'network-checkpoint.yml'

                echo 'Running network-dmvpn.yml...'
                ansiblePlaybook credentialsId: 'scarter-jenkins_key', colorized: true, disableHostKeyChecking: true, inventory: "${env.ANSIBLE_INVENTORY_DIR}/test/wan-testbed.yml", playbook: 'network-dmvpn.yml'

                echo 'Running network-dmvpn-check.yml...'
                ansiblePlaybook credentialsId: 'scarter-jenkins_key', colorized: true, disableHostKeyChecking: true, inventory: "${env.ANSIBLE_INVENTORY_DIR}/test/wan-testbed.yml", playbook: 'network-dmvpn-check.yml'

                echo 'Running network-rollback.yml...'
                ansiblePlaybook credentialsId: 'scarter-jenkins_key', colorized: true, disableHostKeyChecking: true, inventory: "${env.ANSIBLE_INVENTORY_DIR}/test/wan-testbed.yml", playbook: 'network-rollback.yml'
            }
        }
        post {
            success {
                properties([[$class: 'HudsonNotificationProperty', endpoints: [[buildNotes: 'Build Passed', urlInfo: [urlOrId: ' http://cisco-spark-integration-management-ext.cloudhub.io/api/hooks/8fde6043-69b6-11e8-bf37-06c25f4e7996', urlType: 'PUBLIC']]]]])
            }
            failure {
                properties([[$class: 'HudsonNotificationProperty', endpoints: [[buildNotes: 'Build Failed', urlInfo: [urlOrId: ' http://cisco-spark-integration-management-ext.cloudhub.io/api/hooks/8fde6043-69b6-11e8-bf37-06c25f4e7996', urlType: 'PUBLIC']]]]])
            }
            always {
                echo 'Cleaning Workspace...'
                deleteDir()
            }
        }
    }
}
