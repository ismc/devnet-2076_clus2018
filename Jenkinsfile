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
                    userRemoteConfigs: [[credentialsId: 'scarter-jenkins_key', url: 'git@github.com:ismc/netdevops.git']]
                ])
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/master']],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'inventory']],
                    submoduleCfg: [],
                    userRemoteConfigs: [[credentialsId: 'scarter-jenkins_key', url: 'git@github.com:ismc/inventory-test.git']]
                ])
            }
        }
        stage('Destroy Testbed') {
            steps {
                script {
                    try {
                        echo 'Destroying Cloud...'
                        retry(3) {
                            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'Ansible (scarter)']]) {
                                ansiblePlaybook colorized: true, extras: "-e cloud_model=wan-testbed -e cloud_inventory_dir=${env.ANSIBLE_INVENTORY_DIR} -e cloud_instance=wan-testbed -e cloud_project=scarter -e cloud_key_name=scarter-jenkins", playbook: 'cloud/destroy-cloud.yml'
                            }
                        }
                    } catch (e) {
                        echo 'Previous testbed not found!'
                    }
                }
            }
        }
        stage('Build Testbed') {
            steps {
                echo 'Building Cloud...'
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'Ansible (scarter)']]) {
                  ansiblePlaybook colorized: true, extras: "-e cloud_model=wan-testbed -e cloud_inventory_dir=${env.ANSIBLE_INVENTORY_DIR} -e cloud_instance=wan-testbed -e cloud_project=scarter -e cloud_key_name=scarter-jenkins", playbook: 'cloud/build-cloud.yml'
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
                echo 'Wait for the Routers to come up...'
                ansiblePlaybook credentialsId: 'scarter-jenkins_key', colorized: true, limit: 'network', disableHostKeyChecking: true, inventory: "${env.ANSIBLE_INVENTORY_DIR}/wan-testbed.yml", playbook: 'common/check-ssh.yml'

                echo 'Running network-system.yml...'
                ansiblePlaybook credentialsId: 'scarter-jenkins_key', colorized: true, disableHostKeyChecking: true, inventory: "${env.ANSIBLE_INVENTORY_DIR}/wan-testbed.yml", playbook: 'net/network-system.yml'

                echo 'Running network-security.yml...'
                ansiblePlaybook credentialsId: 'scarter-jenkins_key', colorized: true, disableHostKeyChecking: true, inventory: "${env.ANSIBLE_INVENTORY_DIR}/wan-testbed.yml", playbook: 'net/network-security.yml'

                echo 'Running network-interfaces.yml...'
                ansiblePlaybook credentialsId: 'scarter-jenkins_key', colorized: true, disableHostKeyChecking: true, inventory: "${env.ANSIBLE_INVENTORY_DIR}/wan-testbed.yml", playbook: 'net/network-interfaces.yml'

                echo 'Running network-dmvpn.yml...'
                ansiblePlaybook credentialsId: 'scarter-jenkins_key', colorized: true, disableHostKeyChecking: true, inventory: "${env.ANSIBLE_INVENTORY_DIR}/wan-testbed.yml", playbook: 'net/network-dmvpn.yml'

                echo 'Running network-dmvpn-check.yml...'
                ansiblePlaybook credentialsId: 'scarter-jenkins_key', colorized: true, disableHostKeyChecking: true, inventory: "${env.ANSIBLE_INVENTORY_DIR}/wan-testbed.yml", playbook: 'net/network-dmvpn-check.yml'
            }
        }
        stage('Clean Workspace') {
            steps {
                echo 'Cleaning Workspace...'
                deleteDir()
            }
        }
    }
}
