// SPDX-FileCopyrightText: 2023 Zextras <https://www.zextras.com>
//
// SPDX-License-Identifier: AGPL-3.0-only

pipeline {
    agent {
        node {
            label 'base'
        }
    }
    environment {
        LC_ALL="C.UTF-8"
        jenkins_build="true"
    }
    options {
        buildDiscarder(logRotator(numToKeepStr: '25'))
        timeout(time: 30, unit: 'MINUTES')
        skipDefaultCheckout()
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                }
            }
        }
        stage('Build deb/rpm') {
            stages {
                // Replace the pkgrel value with the git commit hash to ensure that
                // each merged PR has unique artifacts and to prevent conflicts between them.
                // Note that the pkgrel value will remain as it was in the codebase to avoid
                // conflicts between multiple open PRs
                stage('Add timestamp and commit hash') {
                    when {
                        branch 'develop'
                    }
                    steps {
                        script {
                            def timestamp = sh(script: 'date +%s', returnStdout: true).trim()
                            def gitCommitShort = env.GIT_COMMIT.take(8)
                            sh """
                                sed -i "s/pkgrel=\\".*\\"/pkgrel=\\"${timestamp}+${gitCommitShort}\\"/" ./package/PKGBUILD
                            """
                        }
                    }
                }
                stage('Stash') {
                    steps {
                        stash includes: "yap.json,package/**", name: 'binaries'
                    }
                }
                stage('yap') {
                    parallel {
                        stage('Ubuntu') {
                            agent {
                                node {
                                    label 'yap-ubuntu-20-v1'
                                }
                            }
                            steps {
                                container('yap') {
                                    unstash 'binaries'
                                    sh 'yap build ubuntu -ds . '
                                    stash includes: 'artifacts/', name: 'artifacts-deb'
                                }
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: "artifacts/*.deb", fingerprint: true
                                }
                            }
                        }
                        stage('RHEL') {
                            agent {
                                node {
                                    label 'yap-rocky-8-v1'
                                }
                            }
                            steps {
                                container('yap') {
                                    unstash 'binaries'
                                    sh 'yap build rocky -ds . '
                                    stash includes: 'artifacts/*.rpm', name: 'artifacts-rpm'
                                }
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: "artifacts/*.rpm", fingerprint: true
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Upload To Develop') {
            when {
                branch 'develop'
            }
            steps {
                unstash 'artifacts-deb'
                unstash 'artifacts-rpm'
                script {
                    def server = Artifactory.server 'zextras-artifactory'
                    def buildInfo
                    def uploadSpec

                    buildInfo = Artifactory.newBuildInfo()
                    uploadSpec = """{
                        "files": [
                            {
                                "pattern": "artifacts/*.deb",
                                "target": "ubuntu-devel/pool/",
                                "props": "deb.distribution=focal;deb.distribution=jammy;deb.distribution=noble;deb.component=main;deb.architecture=amd64;vcs.revision=${env.GIT_COMMIT}"
                            },
                            {
                                "pattern": "artifacts/(carbonio-files-db)-(*).x86_64.rpm",
                                "target": "centos8-devel/zextras/{1}/{1}-{2}.x86_64.rpm",
                                "props": "rpm.metadata.arch=x86_64;rpm.metadata.vendor=zextras;vcs.revision=${env.GIT_COMMIT}"
                            },
                            {
                                "pattern": "artifacts/(carbonio-files-db)-(*).x86_64.rpm",
                                "target": "rhel9-devel/zextras/{1}/{1}-{2}.x86_64.rpm",
                                "props": "rpm.metadata.arch=x86_64;rpm.metadata.vendor=zextras;vcs.revision=${env.GIT_COMMIT}"
                            }
                        ]
                    }"""
                    server.upload spec: uploadSpec, buildInfo: buildInfo, failNoOp: false
                }
            }
        }
        stage('Upload & Promotion Config') {
            when {
                anyOf {
                    branch 'release/*'
                    buildingTag()
                }
            }
            steps {
                unstash 'artifacts-deb'
                unstash 'artifacts-rpm'
                script {
                    def server = Artifactory.server 'zextras-artifactory'
                    def buildInfo
                    def uploadSpec
                    def config

                    //ubuntu
                    buildInfo = Artifactory.newBuildInfo()
                    buildInfo.name += "-ubuntu"
                    uploadSpec = """{
                        "files": [
                            {
                                "pattern": "artifacts/carbonio-files-db*.deb",
                                "target": "ubuntu-rc/pool/",
                                "props": "deb.distribution=focal;deb.distribution=jammy;deb.distribution=noble;deb.component=main;deb.architecture=amd64;vcs.revision=${env.GIT_COMMIT}"
                            }
                        ]
                    }"""
                    server.upload spec: uploadSpec, buildInfo: buildInfo, failNoOp: false
                    config = [
                            'buildName'          : buildInfo.name,
                            'buildNumber'        : buildInfo.number,
                            'sourceRepo'         : 'ubuntu-rc',
                            'targetRepo'         : 'ubuntu-release',
                            'comment'            : 'Do not change anything! Just press the button',
                            'status'             : 'Released',
                            'includeDependencies': false,
                            'copy'               : true,
                            'failFast'           : true
                    ]
                    Artifactory.addInteractivePromotion server: server, promotionConfig: config, displayName: "Ubuntu Promotion to Release"
                    server.publishBuildInfo buildInfo

                    //rhel 8
                    buildInfo = Artifactory.newBuildInfo()
                    buildInfo.name += "-centos8"
                    uploadSpec = """{
                        "files": [
                            {
                                "pattern": "artifacts/(carbonio-files-db)-(*).x86_64.rpm",
                                "target": "centos8-rc/zextras/{1}/{1}-{2}.x86_64.rpm",
                                "props": "rpm.metadata.arch=x86_64;rpm.metadata.vendor=zextras;vcs.revision=${env.GIT_COMMIT}"
                            }
                        ]
                    }"""
                    server.upload spec: uploadSpec, buildInfo: buildInfo, failNoOp: false
                    config = [
                            'buildName'          : buildInfo.name,
                            'buildNumber'        : buildInfo.number,
                            'sourceRepo'         : 'centos8-rc',
                            'targetRepo'         : 'centos8-release',
                            'comment'            : 'Do not change anything! Just press the button',
                            'status'             : 'Released',
                            'includeDependencies': false,
                            'copy'               : true,
                            'failFast'           : true
                    ]
                    Artifactory.addInteractivePromotion server: server, promotionConfig: config, displayName: "RHEL8 Promotion to Release"
                    server.publishBuildInfo buildInfo

                    //rhel 9
                    buildInfo = Artifactory.newBuildInfo()
                    buildInfo.name += "-rhel9"
                    uploadSpec = """{
                        "files": [
                            {
                                "pattern": "artifacts/(carbonio-files-db)-(*).x86_64.rpm",
                                "target": "rhel9-rc/zextras/{1}/{1}-{2}.x86_64.rpm",
                                "props": "rpm.metadata.arch=x86_64;rpm.metadata.vendor=zextras;vcs.revision=${env.GIT_COMMIT}"
                            }
                        ]
                    }"""
                    server.upload spec: uploadSpec, buildInfo: buildInfo, failNoOp: false
                    config = [
                            'buildName'          : buildInfo.name,
                            'buildNumber'        : buildInfo.number,
                            'sourceRepo'         : 'rhel9-rc',
                            'targetRepo'         : 'rhel9-release',
                            'comment'            : 'Do not change anything! Just press the button',
                            'status'             : 'Released',
                            'includeDependencies': false,
                            'copy'               : true,
                            'failFast'           : true
                    ]
                    Artifactory.addInteractivePromotion server: server, promotionConfig: config, displayName: "RHEL9 Promotion to Release"
                    server.publishBuildInfo buildInfo
                }
            }
        }
    }
}
