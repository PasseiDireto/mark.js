@Library("notify-jenkins-library@main") _

pipeline {
    agent {
        label 'nodejs-build-node-cd'
    }
    environment {
        GIT_REPOSITORY_NAME = 'pd-mark'
        CHANNEL = credentials('channel-content-discovery-team')
        COMMIT_AUTHOR = sh(
            script: 'git log -n 1 | grep Author',
            returnStdout: true
        ).trim()
    }
    stages {
        stage('Setup') {
            steps {
                sh 'sudo n 14'
            }
        }
        stage('Build') {
            steps {
                sh 'npm install'
            }
        }
        stage('Increment version') {
            when {
                anyOf {
                    branch 'master';
                    branch 'release/*'
                }
            }
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        keyFileVariable: 'GIT_KEY',
                        credentialsId: 'pd-github-ssh-key'
                    )
                ]) {
                    sh '''
                        # configure user
                        git config push.default simple
                        git config user.email "tecnologia@passeidireto.com"
                        git config user.name "Jenkins"
                        # updating version's minor
                        npm version minor -m "version bump by Jenkins"
                        # start SSH agent
                        eval "$(ssh-agent -s)"
                        ssh-add ${GIT_KEY}
                        # push new version
                        git push --set-upstream origin ${BRANCH_NAME}
                        # kill SSH agent
                        ssh-agent -k
                    '''
                }
            }
        }

        stage('SONARQUBE') {
            when {
                branch 'master'
            }
            environment {
                SONAR_SCANNER_HOME = tool 'LocalSonarScanner'
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        sh "${SONAR_SCANNER_HOME}/bin/sonar-scanner"
                    """
                }
            }
        }

        stage('Publish') {
            when {
                anyOf {
                    branch 'master';
                    branch 'release/*'
                }
            }
            steps {
                sh '''
                    npm install --production
                    npm prune --production
                    npm publish
                '''
            }
        }
        stage ('Merge modifications') {
            when {
                branch 'release/*'
            }
            environment {
                RELEASE_NUMBER = sh(
                    script: 'echo ${BRANCH_NAME} | sed -e "s|.*release/||"',
                    returnStdout: true
                ).trim()
            }
            steps {
                build \
                    job: '/merge-back-to-master',
                    parameters: [
                        string(name: 'RELEASE_NUMBER', value: "${RELEASE_NUMBER}"),
                        string(name: 'GIT_REPOSITORY_NAME', value: "${GIT_REPOSITORY_NAME}")
                    ]
            }
        }
    }
    post {
        success {
            notify(
			"${CHANNEL}",
			"Latest status of build [#${env.BUILD_NUMBER}](${env.BUILD_URL})",
			"28a745",
			[	[name: "Status", template: "Build Success"],
				[name: "Branch", template: "${env.GIT_BRANCH}"],
				[name: "Commit", template: "${env.GIT_COMMIT}"]
			]
			)
        }
        failure {
            notify(
			"${CHANNEL}",
			"Latest status of build [#${env.BUILD_NUMBER}](${env.BUILD_URL})",
			"dc3545",
			[	[name: "Status", template: "Build Failed"],
				[name: "Branch", template: "${env.GIT_BRANCH}"],
				[name: "Commit", template: "${env.GIT_COMMIT}"]
			]
			)
        }
    }
}