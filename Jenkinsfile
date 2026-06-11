pipeline {
    agent any

    parameters {
        choice(
            name: 'ENV',
            choices: ['dev', 'staging', 'prod'],
            description: '部署环境'
        )
        string(
            name: 'APP_VERSION',
            defaultValue: '',
            description: '自定义版本号（留空自动生成）'
        )
    }

    environment {
        HARBOR_URL = "13.229.80.117:8081"
        HARBOR_CRED = credentials('HARBOR-ACCOUNT')
        GIT_TOKEN = credentials('3b16e291-f7f9-4b0e-b6a0-5ee25df70bc5')
        HELM_REPO = "https://github.com/BubbleTea1234/online-boutique-helm.git"

        DEV_BRANCH     = "develop"
        STAGING_BRANCH = "release/staging"
        PROD_BRANCH    = "main"
    }

    stages {
        stage('Init') {
            steps {
                script {
                    GIT_HASH = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()

                    // ================ 改这一行 ================
                    env.APP_VERSION = params.APP_VERSION ?: "build-${BUILD_NUMBER}-${GIT_HASH}"

                    switch (params.ENV) {
                        case 'prod':
                            env.HELM_BRANCH = PROD_BRANCH
                            env.TARGET_NAMESPACE = "boutique-prod"
                            break
                        case 'staging':
                            env.HELM_BRANCH = STAGING_BRANCH
                            env.TARGET_NAMESPACE = "boutique-staging"
                            break
                        default:
                            env.HELM_BRANCH = DEV_BRANCH
                            env.TARGET_NAMESPACE = "boutique-dev"
                    }
                    echo "环境: ${params.ENV}  版本: ${APP_VERSION}  Helm分支: ${HELM_BRANCH}"
                }
            }
        }

        stage('Build Images') {
            parallel {
                stage('frontend') { steps { dir('frontend') { buildImage('frontend') } } }
                stage('cartservice') { steps { dir('cartservice') { buildImage('cartservice') } } }
                stage('productcatalog') { steps { dir('productcatalogservice') { buildImage('productcatalogservice') } } }
                stage('currencyservice') { steps { dir('currencyservice') { buildImage('currencyservice') } } }
                stage('paymentservice') { steps { dir('paymentservice') { buildImage('paymentservice') } } }
                stage('shippingservice') { steps { dir('shippingservice') { buildImage('shippingservice') } } }
                stage('emailservice') { steps { dir('emailservice') { buildImage('emailservice') } } }
                stage('checkoutservice') { steps { dir('checkoutservice') { buildImage('checkoutservice') } } }
                stage('recommendationservice') { steps { dir('recommendationservice') { buildImage('recommendationservice') } } }
                stage('adservice') { steps { dir('adservice') { buildImage('adservice') } } }
            }
        }

        stage('Update Helm Chart') {
            steps {
                dir('helm-work') {
                    sh """
                        rm -rf .git
                        git init
                        git remote add origin ${HELM_REPO}
                        git fetch origin ${HELM_BRANCH}
                        git checkout ${HELM_BRANCH}

                        git config user.email "jenkins@boutique.com"
                        git config user.name "Jenkins CI"

                        VALUES="values.yaml"
                        sed -i "s|tag: latest|tag: ${APP_VERSION}|" \${VALUES}

                        echo "version: ${APP_VERSION}" > .version
                        echo "env: ${params.ENV}" >> .version

                        git add -A
                        git commit -m "[${params.ENV}] update to ${APP_VERSION}"
                        git push https://\${GIT_TOKEN_USR}:\${GIT_TOKEN_PSW}@github.com/BubbleTea1234/online-boutique-helm.git \${HELM_BRANCH}
                    """
                }
            }
        }
    }

    post {
        always { sh 'docker logout ${HARBOR_URL}' }
        success { echo "✅ ${params.ENV} 部署完成" }
    }
}

def buildImage(serviceName) {
    sh """
        docker build -t ${HARBOR_URL}/${params.ENV}/${serviceName}:${APP_VERSION} .
        docker login ${HARBOR_URL} -u ${HARBOR_CRED_USR} -p ${HARBOR_CRED_PSW}
        docker push ${HARBOR_URL}/${params.ENV}/${serviceName}:${APP_VERSION}
    """
}
