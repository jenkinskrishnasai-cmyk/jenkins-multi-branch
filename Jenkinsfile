pipeline {
    agent any

    parameters {
        booleanParam(name: 'MAKE_UNSTABLE', defaultValue: false,
                     description: 'Failing tests = UNSTABLE (yellow) instead of FAILURE (red)')
    }

    environment {
        NODE_EXTRA_CA_CERTS = "${WORKSPACE}/corp-root.crt"
        npm_config_cache    = "${WORKSPACE}/.npm"
    }

    stages {
        // Like apheleia's pom.xml: decide the version from the branch or tag
        stage('Info') {
            steps {
                script {
                    def version = sh(script: "grep -m1 '\"version\"' package.json | cut -d '\"' -f4",
                                     returnStdout: true).trim()
                    if (env.TAG_NAME?.startsWith('release-')) {
                        env.IMAGE_TAG = env.TAG_NAME.substring(8)                  // release-1.0.0 -> 1.0.0
                    } else {
                        def b = (env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'local').replaceAll('.*/', '')
                        if (b.length() > 20) { b = b.substring(0, 20) }
                        env.IMAGE_TAG = "${version}-SNAPSHOT-${b}"                 // 1.0.0-SNAPSHOT-main
                    }
                }
                echo "BRANCH_NAME=${env.BRANCH_NAME}  TAG_NAME=${env.TAG_NAME}"
                echo "Image would be: jenkins-multibranch-demo:${env.IMAGE_TAG}"
            }
        }

        stage('Prepare') {
            steps { sh 'cp /usr/local/share/ca-certificates/corp-root.crt .' }
        }

        stage('Install') {
            agent { docker { image 'node:14-alpine'; reuseNode true } }
            steps { sh 'npm install --no-audit --no-fund' }
        }

        stage('Unit tests') {
            agent { docker { image 'node:14-alpine'; reuseNode true } }
            steps {
                script {
                    if (params.MAKE_UNSTABLE) {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') { sh 'npm test' }
                    } else {
                        sh 'npm test'
                    }
                }
            }
            post {
                always { junit testResults: 'reports/junit.xml', allowEmptyResults: true }
            }
        }

        stage('Snapshot image') {
            when { not { buildingTag() } }
            steps { echo "docker build + push ${env.IMAGE_TAG} -> SNAPSHOT registry (simulated)" }
        }

        stage('Release image') {
            when { tag 'release-*' }
            steps { echo "docker build + push ${env.IMAGE_TAG} -> RELEASE registry (simulated)" }
        }

        stage('Deploy to test') {
            when { branch 'main' }
            steps { echo 'Deploying main to the test server (simulated)' }
        }
    }
}
