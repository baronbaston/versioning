String version = ''
String gitCommit = ''
String branchName = ''
String projectName = 'versioning'

boolean pr = false

pipeline {

    agent {
        docker {
            image 'openjdk:8'
            label 'docker'
        }
    }

    options {
        // General Jenkins job properties
        buildDiscarder(logRotator(numToKeepStr: '40'))
        // Timestamps
        timestamps()
        // No durability
        durabilityHint('PERFORMANCE_OPTIMIZED')
    }

    stages {

        stage('Setup') {
            steps {
                script {
                    branchName = ontrackBranchName(BRANCH_NAME)
                    echo "Ontrack branch name = ${branchName}"
                    pr = BRANCH_NAME ==~ 'PR-.*'
                }
                script {
                    if (pr) {
                        echo "No Ontrack setup for PR."
                    } else {
                        echo "Ontrack setup for ${branchName}"
                        ontrackBranchSetup(project: projectName, branch: branchName, script: """
                            branch.config {
                                gitBranch '${branchName}', [
                                    buildCommitLink: [
                                        id: 'git-commit-property'
                                    ]
                                ]
                            }
                        """)
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh '''\
#!/bin/bash
set -e

./gradlew \\
    clean \\
    versionDisplay \\
    build \\
    --stacktrace \\
    --profile \\
    --parallel \\
    --console plain
'''
            }
        }

        stage('Publication') {
            when {
                branch 'release/*'
            }
            environment {
                BINTRAY = credentials('BINTRAY')
            }
            steps {
                sh '''\
#!/bin/bash
set -e

./gradlew \\
    versionDisplay \\
    versionFile \\
    publishPluginToBintray \\
    -x test \\
    --stacktrace \\
    --profile \\
    --console plain \\
    -PbintrayUser=${BINTRAY_USR} \\
    -PbintrayApiKey=${BINTRAY_PSW}
'''
                script {
                    // Reads version information
                    def props = readProperties(file: 'build/version.properties')
                    version = props.VERSION_DISPLAY
                    gitCommit = props.VERSION_COMMIT
                }
                echo "Version = ${version}"
            }
        }

        stage('Release') {
            when {
                branch 'release/*'
            }
            environment {
                GIT_COMMIT = "${gitCommit}"
                VERSION = "${version}"
                GITHUB = credentials('GITHUB_NEMEROSA_JENKINS2')
            }
            steps {
                sh '''\
#!/bin/bash
set -e

curl -X POST "https://api.github.com/repos/nemerosa/versioning/releases" \\
    --data "{\"target_commitish\":\"${GIT_COMMIT}\",\"tag_name\":\"${VERSION}\",\"name\":\"${VERSION}\"}" \\
    --user '${GITHUB}'
'''
            }
        }
    }
}