#!groovy
pipeline {
	agent {
		node {
			label 'linuxSecondary'
			//customWorkspace '/some/other/path'
		}
	}
    environment {
        REPO_NAME = 'scmDockerDemo'
		LOD = '1.0.0'
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
        SVNURL = 'http://svn.maximus.com/svn/'
        JAVA_OPTS='-Xmx1024m -Xms512m'
    }
    options {
        buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '10', numToKeepStr: '20'))
        timestamps()
        //retry(3)
        timeout time:10, unit:'MINUTES'
    }
	parameters {
        string(defaultValue: "develop", description: 'Branch Specifier', name: 'SPECIFIER')
        booleanParam(defaultValue: false, description: 'Deploy to QA Environment ?', name: 'DEPLOY_QA')
        booleanParam(defaultValue: false, description: 'Deploy to UAT Environment ?', name: 'DEPLOY_UAT')
        booleanParam(defaultValue: false, description: 'Deploy to PROD Environment ?', name: 'DEPLOY_PROD')
    }
    stages {
        stage("Initialize") {
            steps {
                script {
                    notifyBuild('STARTED')
                    echo "${BUILD_NUMBER} - ${env.BUILD_ID} on ${env.JENKINS_URL}"
                    echo "Branch Specifier :: ${params.SPECIFIER}"
                    echo "Deploy to QA? :: ${params.DEPLOY_QA}"
                    echo "Deploy to UAT? :: ${params.DEPLOY_UAT}"
                    echo "Deploy to PROD? :: ${params.DEPLOY_PROD}"
                }
            }
        }
        stage('Checkout') {
            steps {
                echo 'Checkout Repo'
                checkout([$class: 'SubversionSCM', additionalCredentials: [], browser: [$class: 'CollabNetSVN', url: 'http://svn.maximus.com/viewvc/scmDockerDemo/'], excludedCommitMessages: '', excludedRegions: '/snapshot/*', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[credentialsId: '', depthOption: 'infinity', ignoreExternalsOption: true, local: '.', remote: 'http://svn.maximus.com/svn/scmDockerDemo/lods/dev_1.0.0']], workspaceUpdater: [$class: 'CheckoutUpdater']])
            }
        }
        stage('Build') {
            steps {
				build job: 'SCM/SCM_dockerDemo-scmScript/', parameters: [string(name: 'SNAPSHOT', value: '')]
            }
        }
        stage('Static Code Coverage Analysis') {
            parallel {
              stage('Execute Whitesource Analysis') {
                  steps {
                      whitesource jobApiToken: '', jobCheckPolicies: 'global', jobForceUpdate: 'global', libExcludes: '', libIncludes: '', product: "$WS_PRODUCT_TOKEN", productVersion: '', projectToken: "$WS_PROJECT_TOKEN", requesterEmail: ''
                  }
              }
              stage('SonarQube analysis') {
                  steps {
                      build 'SCM/scmDockerDemo_sonarqube'
                  }
              }
            }
        }
         stage('Docker Tag & Push') {
             steps {
                 script {
                     branchName = getCurrentBranch()
                     shortCommitHash = getShortCommitHash()
                     IMAGE_VERSION = "${BUILD_NUMBER}-" + branchName + "-" + shortCommitHash
                     sh 'eval $(aws ecr get-login --no-include-email --region us-east-1)'
                     sh "docker-compose build"
                     sh "docker tag ${REPOURL}/${APP_NAME}:latest ${REPOURL}/${APP_NAME}:${IMAGE_VERSION}"
                     sh "docker push ${REPOURL}/${APP_NAME}:${IMAGE_VERSION}"
                     sh "docker push ${REPOURL}/${APP_NAME}:latest"

                     sh "docker rmi ${REPOURL}/${APP_NAME}:${IMAGE_VERSION} ${REPOURL}/${APP_NAME}:latest"
                 }
             }
         }

        stage('Deploy - CI') {
            steps {
                echo "Deploying to CI Environment."
				input id: 'UDUCP', message: 'Update scmdockerdemo service?', parameters: [[$class: 'DynamicReferenceParameter', choiceType: 'ET_TEXT_BOX', description: '', name: 'SNAPSHOT', omitValueField: false, randomName: 'choice-parameter-18754605303716994', referencedParameters: 'PROJECT', script: [$class: 'ScriptlerScript', parameters: [PROJECT: 'scmDockerDemo', '': ''], scriptlerScriptId: 'SNAPSHOT.groovy']]]
            }
        }

        stage('Deploy - QA') {
            when {
                expression {
                    params.DEPLOY_QA == true
                }
            }
            steps {
                echo "Deploy to QA..."
            }
        }
        stage('Deploy - UAT') {
            when {
                expression {
                    params.DEPLOY_UAT == true
                }
            }
            steps {
                echo "Deploy to UAT..."
            }
        }
        stage('Deploy - Production') {
            when {
                expression {
                    params.DEPLOY_PROD == true
                }
            }
            steps {
                echo "Deploy to PROD..."
            }
        }
    }

    post {
        /*
         * These steps will run at the end of the pipeline based on the condition.
         * Post conditions run in order regardless of their place in pipeline
         * 1. always - always run
         * 2. changed - run if something changed from last run
         * 3. aborted, success, unstable or failure - depending on status
         */
        always {
            echo "I AM ALWAYS first"
            notifyBuild("${currentBuild.currentResult}")
        }
        aborted {
            echo "BUILD ABORTED"
        }
        success {
            echo "BUILD SUCCESS"
            echo "Keep Current Build If branch is master"
//            keepThisBuild()
        }
        unstable {
            echo "BUILD UNSTABLE"
        }
        failure {
            echo "BUILD FAILURE"
        }
    }
}
def getCurrentSnapshot() {
    return sh(returnStdout: true, script: "/usr/bin/java -Xmx512m -Xms256m -cp /opt/cie/bin/scmScript.jar:/opt/cie/bin/lib/*.jar com.maximus.scm.GetLatestSnapshot -r scmDockerDemo -l dev_1.0.0").trim()
}

def keepThisBuild() {
    currentBuild.setKeepLog(true)
    currentBuild.setDescription("Test Description")
}

def getShortCommitHash() {
    return sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
}

def getChangeAuthorName() {
    return sh(returnStdout: true, script: "git show -s --pretty=%an").trim()
}

def getChangeAuthorEmail() {
    return sh(returnStdout: true, script: "git show -s --pretty=%ae").trim()
}

def getChangeSet() {
    return sh(returnStdout: true, script: 'git diff-tree --no-commit-id --name-status -r HEAD').trim()
}

def getChangeLog() {
    return sh(returnStdout: true, script: "git log --date=short --pretty=format:'%ad %aN <%ae> %n%n%x09* %s%d%n%b'").trim()
}

def getCurrentBranch () {
    return sh (
            script: 'git rev-parse --abbrev-ref HEAD',
            returnStdout: true
    ).trim()
}

def isPRMergeBuild() {
    return (env.BRANCH_NAME ==~ /^PR-\d+$/)
}

def notifyBuild(String buildStatus = 'STARTED') {
    // build status of null means successful
    buildStatus = buildStatus ?: 'SUCCESS'

    def branchName = getCurrentBranch()
    def shortCommitHash = getShortCommitHash()
    def changeAuthorName = getChangeAuthorName()
    def changeAuthorEmail = getChangeAuthorEmail()
    def changeSet = getChangeSet()
    def changeLog = getChangeLog()

    // Default values
    def colorName = 'RED'
    def colorCode = '#FF0000'
    def subject = "${buildStatus}: '${env.JOB_NAME} [${env.BUILD_NUMBER}]'" + branchName + ", " + shortCommitHash
    def summary = "Started: Name:: ${env.JOB_NAME} \n " +
            "Build Number: ${env.BUILD_NUMBER} \n " +
            "Build URL: ${env.BUILD_URL} \n " +
            "Short Commit Hash: " + shortCommitHash + " \n " +
            "Branch Name: " + branchName + " \n " +
            "Change Author: " + changeAuthorName + " \n " +
            "Change Author Email: " + changeAuthorEmail + " \n " +
            "Change Set: " + changeSet

    if (buildStatus == 'STARTED') {
        color = 'YELLOW'
        colorCode = '#FFFF00'
    } else if (buildStatus == 'SUCCESS') {
        color = 'GREEN'
        colorCode = '#00FF00'
    } else {
        color = 'RED'
        colorCode = '#FF0000'
    }

    // Send notifications
	slackSend (baseUrl: 'https://maximusworkspace.slack.com/services/hooks/jenkins-ci/', 
	channel: 'scmdockerdemo', 
	color: color, 
	message: '${currentSNAPSHOT} is ' + summary, 
	tokenCredentialId: 'jenkins-slack-integration')

    if (buildStatus == 'FAILURE') {
        emailext attachLog: true, body: summary, compressLog: true, recipientProviders: [brokenTestsSuspects(), brokenBuildSuspects(), culprits()], replyTo: 'noreply@yourdomain.com', subject: subject, to: 'mpatel@yourdomain.com'
    }
}
