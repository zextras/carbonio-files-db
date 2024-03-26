// SPDX-FileCopyrightText: 2022 Zextras <https://www.zextras.com>
//
// SPDX-License-Identifier: AGPL-3.0-only

pipeline {
    agent {
        node {
            label 'carbonio-agent-v1'
        }
    }
    environment {
        LC_ALL="C.UTF-8"
        jenkins_build="true"
    }
    parameters {
        booleanParam defaultValue: false, description: 'Whether to upload the packages in playground repositories', name: 'PLAYGROUND'
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
            }
        }
        stage('Build deb/rpm') {
            stages {
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
                                    label 'yap-agent-ubuntu-20.04-v2'
                                }
                            }
                            steps {
                                dir('/tmp/staging') {
                                  unstash 'binaries'
                                }
                                sh 'sudo yap build ubuntu /tmp/staging/'
                                stash includes: 'artifacts/', name: 'artifacts-deb'
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
                                    label 'yap-agent-rocky-8-v2'
                                }
                            }
                            steps {
                                dir('/tmp/staging') {
                                  unstash 'binaries'
                                }
                                sh 'sudo yap build rocky /tmp/staging/'
                                stash includes: 'artifacts/x86_64/*.rpm', name: 'artifacts-rpm'
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: "artifacts/x86_64/*.rpm", fingerprint: true
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
                script {
                    def server = Artifactory.server 'zextras-artifactory'
                    def buildInfo
                    def uploadSpec

                    buildInfo = Artifactory.newBuildInfo()
                    uploadSpec = '''{
                        "files": [
                            {
                                "pattern": "artifacts/*.deb",
                                "target": "ubuntu-devel/pool/",
                                "props": "deb.distribution=focal;deb.distribution=jammy;deb.component=main;deb.architecture=amd64"
                            },
                            {
                                "pattern": "artifacts/x86_64/(carbonio-files-db)-(*).x86_64.rpm",
                                "target": "centos8-devel/zextras/{1}/{1}-{2}.x86_64.rpm",
                                "props": "rpm.metadata.arch=x86_64;rpm.metadata.vendor=zextras"
                            },
                            {
                                "pattern": "artifacts/x86_64/(carbonio-files-db)-(*).x86_64.rpm",
                                "target": "rhel9-devel/zextras/{1}/{1}-{2}.x86_64.rpm",
                                "props": "rpm.metadata.arch=x86_64;rpm.metadata.vendor=zextras"
                            }
                        ]
                    }'''
                    server.upload spec: uploadSpec, buildInfo: buildInfo, failNoOp: false
                }
            }
        }
         stage('Upload To Playground') {
            when {
                anyOf {
                    branch 'playground/*'
                    expression { params.PLAYGROUND == true }
                }
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
                                "pattern": "artifacts/carbonio-files-db*.deb",
                                "target": "ubuntu-playground/pool/",
                                "props": "deb.distribution=focal;deb.distribution=jammy;deb.component=main;deb.architecture=amd64"
                            },
                            {
                                "pattern": "artifacts/x86_64/(carbonio-files-db)-(*).x86_64.rpm",
                                "target": "centos8-playground/zextras/{1}/{1}-{2}.x86_64.rpm",
                                "props": "rpm.metadata.arch=x86_64;rpm.metadata.vendor=zextras"
                            },
                            {
                                "pattern": "artifacts/x86_64/(carbonio-files-db)-(*).x86_64.rpm",
                                "target": "rhel9-playground/zextras/{1}/{1}-{2}.x86_64.rpm",
                                "props": "rpm.metadata.arch=x86_64;rpm.metadata.vendor=zextras"
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
                    uploadSpec= """{
                        "files": [
                            {
                                "pattern": "artifacts/carbonio-files-db*.deb",
                                "target": "ubuntu-rc/pool/",
                                "props": "deb.distribution=focal;deb.distribution=jammy;deb.component=main;deb.architecture=amd64"
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
                    uploadSpec= """{
                        "files": [
                            {
                                "pattern": "artifacts/x86_64/(carbonio-files-db)-(*).x86_64.rpm",
                                "target": "centos8-rc/zextras/{1}/{1}-{2}.x86_64.rpm",
                                "props": "rpm.metadata.arch=x86_64;rpm.metadata.vendor=zextras"
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
                    uploadSpec= """{
                        "files": [
                            {
                                "pattern": "artifacts/x86_64/(carbonio-files-db)-(*).x86_64.rpm",
                                "target": "rhel9-rc/zextras/{1}/{1}-{2}.x86_64.rpm",
                                "props": "rpm.metadata.arch=x86_64;rpm.metadata.vendor=zextras"
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

