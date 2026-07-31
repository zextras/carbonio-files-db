// SPDX-FileCopyrightText: 2023 Zextras <https://www.zextras.com>
//
// SPDX-License-Identifier: AGPL-3.0-only

library(
    identifier: 'jenkins-lib-common@feat/verify-only',
    retriever: modernSCM([
        $class: 'GitSCMSource',
        credentialsId: 'jenkins-integration-with-github-account',
        remote: 'git@github.com:zextras/jenkins-lib-common.git',
    ])
)

dt3_pipeline(
    repoName: 'carbonio-files-db',
    packaging: [
        buildFlags: '-ds',
        rockySinglePkg: false,
        ubuntuSinglePkg: false,
    ],
    docker: [[
        dockerfile: 'docker/files-db-sidecar/Dockerfile',
        imageName: 'carbonio-files-db-sidecar',
        platforms: ['linux/amd64', 'linux/arm64'] as Set,
        title: 'Carbonio Files DB Sidecar',
        description: 'Carbonio Files DB sidecar service',
    ]],
    reuse: [projectType: 'CE'],
)
