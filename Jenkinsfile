// Example Jenkinsfile with SIG Security Scan that implements:
// - Black Duck INTELLIGENT & Coverity FULL scans on pushes to "important" branches
// - Black Duck RAPID & Coverity INCREMENTAL scans on PRs to "important" branches
pipeline {
    agent any
    environment {
        // production branches on which we want security reports
        PRODUCTION = "${env.BRANCH_NAME ==~ /^(stage|release)$/ ? 'true' : 'false'}"
        // full scan on pushes to important branches
        FULLSCAN = "${env.BRANCH_NAME ==~ /^(main|master|develop|stage|release)$/ ? 'true' : 'false'}"
        // PR scan on pulll requests to important branches
        PRSCAN = "${env.CHANGE_TARGET ==~ /^(main|master|develop|stage|release)$/ ? 'true' : 'false'}"
        // set project name to be repo name
        PROJECT = sh(script: "basename $GIT_URL .git", returnStdout: true).trim()
	// Coverity Connect server
	CONNECT = 'https://poc329.coverity.synopsys.com'
        COVERITY_TOOL_HOME = "$JENKINS_HOME/tools/cov-analysis-linux64-2023.3.2"
    }
    tools {
        maven 'maven-3.9'
        jdk 'openjdk-11'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn -B compile'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn -B test'
            }
        }
        stage('Scan') {
            parallel {
                stage('Black Duck Full Scan') {
                    when { environment name: 'FULLSCAN', value: 'true' }
                    environment {
                        DETECT_PROJECT_NAME = "$PROJECT"
                        DETECT_PROJECT_VERSION_NAME = "$BRANCH_NAME"
                        DETECT_CODE_LOCATION_NAME = "$PROJECT-$BRANCH_NAME"
                        DETECT_RISK_REPORT_PDF = "${env.PRODUCTION == 'true' ? 'true' : 'false'}"
                    }
                    steps {
                        script {
                            status = synopsys_detect detectProperties: "--detect.policy.check.fail.on.severities=BLOCKER", returnStatus: true
                            if (status == 3) { unstable 'Policy Violation' }
                            else if (status != 0) { error 'Detect Failure' }
                        }
                    }
                }
                stage('Black Duck PR Scan') {
                    when { environment name: 'PRSCAN', value: 'true' }
                    environment {
                        DETECT_PROJECT_NAME = "$PROJECT"
                        DETECT_PROJECT_VERSION_NAME = "$CHANGE_TARGET"
                        DETECT_CODE_LOCATION_NAME = "$PROJECT-$CHANGE_TARGET"
                    }
                    steps {
                        script {
                            status = synopsys_detect detectProperties: "--detect.blackduck.scan.mode=RAPID", returnStatus: true
                            if (status == 3) { unstable 'Policy Violation' }
                            else if (status != 0) { error 'Detect Failure' }
                        }
                    }
                }
                stage('Coverity Full Scan') {
                    when { environment name: 'FULLSCAN', value: 'true' }
                    steps {
                        withCoverityEnvironment(coverityInstanceUrl: "$CONNECT", projectName: "$PROJECT", streamName: "$PROJECT-$BRANCH_NAME") {
                            sh "coverity scan -o commit.connect.url=$COV_URL -o commit.connect.project=$COV_PROJECT -o commit.connect.stream=$COV_STREAM -o commit.connect.description=$BUILD_TAG"
                            script {
                                count = coverityIssueCheck viewName: 'Outstanding Issues', returnIssueCount: true
                                if (count != 0) { unstable 'Issues Detected' }
                            }
                        }
                    }
                }
                stage('Coverity PR Scan - PFI') {
                    when { environment name: 'PRSCAN', value: 'true' }
                    steps {
                        withCoverityEnvironment(coverityInstanceUrl: "$CONNECT", projectName: "$PROJECT", streamName: "$PROJECT-$CHANGE_TARGET") {
                            script {
                                status = sh returnStatus: true, script: """
                                    coverity scan -o commit.connect.url=$COV_URL -o commit.connect.project=$COV_PROJECT -o commit.connect.stream=$COV_STREAM -o commit.connect.comparison-report=comparison-report.json
                                    cat comparison-report.json | jq '.issues[] | select(.presentInReferenceSnapshot == false and (.impact == "Medium" or .impact == "High"))' > new-issues.json
                                    if [ -s new-issues.json ]; then cat new-issues.json | jq; exit 3; fi
                                """
                                if (status == 3) { unstable 'New Issues Detected' }
                                else if (status != 0) { error 'Coverity Failure' }
                            }
                        }
                    }
                }
                stage('Coverity PR Scan - HFI') {
                    when { environment name: 'PRSCAN', value: 'true' }
                    steps {
                        withCoverityEnvironment(coverityInstanceUrl: "$CONNECT", projectName: "$PROJECT", streamName: "$PROJECT-$CHANGE_TARGET") {
                            script {
                                status = sh returnStatus: true, script: """
				    export FILELIST=\$(git --no-pager diff origin/$CHANGE_TARGET --name-only)
                                    env | sort
                                    cov-run-desktop --dir idir --url $COV_URL --stream $COV_STREAM --build mvn -B -DskipTests package
                                    cov-run-desktop --dir idir --url $COV_URL --stream $COV_STREAM --present-in-reference false --ignore-uncapturable-inputs true --text-output issues.txt $FILELIST
                                    if [ -s issues.txt ]; then cat issues.txt; exit 3; fi
                                """
                                if (status == 3) { unstable 'New Issues Detected' }
                                else if (status != 0) { error 'Coverity Failure' }
                            }
                        }
                    }
                }
            }
        }
        stage('Deploy') {
            when { environment name: 'PRODUCTION', value: 'true' }
            steps {
                sh 'mvn -B install'
            }
        }
    }
    post {
        always {
            archiveArtifacts allowEmptyArchive: true, artifacts: 'idir/build-log.txt, idir/output/analysis-log.txt, *_BlackDuck_RiskReport.pdf'
            cleanWs()
        }
    }
}
