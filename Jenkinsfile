pipeline {
    agent any
    tools {
        maven 'maven-3.9'
        jdk 'openjdk-11'
    }
    environment {
        // full scan on pushes to important branches
        FULLSCAN = "${env.BRANCH_NAME ==~ /^(main|master|develop|stage|release)$/ ? 'true' : 'false'}"
        // PR scan on pulll requests to important branches
        PRSCAN = "${env.CHANGE_TARGET ==~ /^(main|master|develop|stage|release)$/ ? 'true' : 'false'}"
        // set project name to be repo name
        PROJECT = sh(script: "basename $GIT_URL .git", returnStdout: true).trim()
        // Coverity Connect server
        CONNECT = 'https://poc329.coverity.synopsys.com'
		BRIDGE_COVERITY_LOCAL = 'true'
		BRIDGE_COVERITY_INSTALL_DIRECTORY = '/opt/coverity/analysis/2023.6.1'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn -B package'
            }
        }
        stage('Coverity Full Scan') {
            when { environment name: 'FULLSCAN', value: 'true' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'poc329.coverity.synopsys.com', usernameVariable: 'COV_USER', passwordVariable: 'COVERITY_PASSPHRASE')]) {
                    script {
                        status = sh returnStatus: true, script: """
                            curl -fLsS -o bridge.zip $BRIDGECLI_LINUX64 && unzip -qo -d $WORKSPACE_TMP bridge.zip && rm -f bridge.zip
                            $WORKSPACE_TMP/synopsys-bridge --verbose --stage connect \
                                coverity.connect.url=$CONNECT \
                                coverity.connect.user.name=$COV_USER \
                                coverity.connect.user.password=$COVERITY \
                                coverity.connect.project.name=$PROJECT \
                                coverity.connect.stream.name=$PROJECT-$BRANCH_NAME \
                                coverity.connect.policy.view="Outstanding Issues"
                        """
                        if (status == 8) { unstable 'policy violation' }
                        else if (status != 0) { error 'scan failure' }
                    }
                }
            }
        }
        stage('Coverity PR Scan') {
                when { environment name: 'PRSCAN', value: 'true' }
                steps {
                withCredentials([usernamePassword(credentialsId: 'poc329.coverity.synopsys.com', usernameVariable: 'COV_USER', passwordVariable: 'COVERITY_PASSPHRASE')]) {
                    script {
                        status = sh returnStatus: true, script: """
                            curl -fLsS -o bridge.zip $BRIDGECLI_LINUX64 && unzip -qo -d $WORKSPACE_TMP bridge.zip && rm -f bridge.zip
                            $WORKSPACE_TMP/synopsys-bridge --verbose --stage connect \
                                coverity.connect.url=$CONNECT \
                                coverity.connect.user.name=$COV_USER \
                                coverity.connect.user.password=$COVERITY \
                                coverity.connect.project.name=$PROJECT \
                                coverity.connect.stream.name=$PROJECT-$CHANGE_TARGET \
                                coverity.automation.prcomment='true' \
                                github.repository.name=$PROJECT \
                                github.repository.branch.name=$BRANCH_NAME \
                                github.repository.owner.name=$CHANGE_AUTHOR \
                                github.repository.pull.number=$CHANGE_ID \
                                github.user.token=$GITHUB_TOKEN
                        """
                        if (status == 8) { unstable 'policy violation' }
                        else if (status != 0) { error 'scan failure' }
                    }
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts allowEmptyArchive: true, artifacts: 'idir/build-log.txt, idir/output/analysis-log.txt'
            cleanWs()
        }
    }
}
