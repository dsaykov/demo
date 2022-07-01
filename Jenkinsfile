@Library(['shared_library@master']) _

PHOTON = 'PHOTON'
GRAVITON = 'GRAVITON'
HADRON = 'HADRON'
MUON = 'MUON'
ELECTRON = 'ELECTRON'
NEUTRON = 'NEUTRON'
PROTON = 'PROTON'
NOWHERE = 'NOWHERE'

DEPLOYMENT_ENV = NOWHERE
ACCOUNT_VALUE = NOWHERE

pipeline {
    environment {
        MAVEN_VERSION = '3.6-jdk-11'
        PROJECT = 'demo'
        DOCKER_IMAGE_NAME = "097002022524.dkr.ecr.eu-west-1.amazonaws.com/${PROJECT}:${GIT_COMMIT}"
      
        HEALTH_CHECK_ENDPOINT = "/health"
        APP_PATH = "/*"
        GIT_TAG = "${TAG_NAME}"
        FARGATE_CPU = "512"
        FARGATE_MEMORY = "1024"
    
        SECRETS_VARS = '[' +
                '{name="spring.datasource.host", source="master-fqdn"}, ' +
                '{name="spring.datasource.port", source="master-port"}, ' +
                '{name="spring.datasource.username", source="master-username"}, ' +
                '{name="spring.datasource.password", source="master-password"}]'
        ENV_VARS = '[' +
                '{name="SERVICES_BASE_URL", value="https://api.__ENV__.paris.rp-cloudinfra.com"}, ' +
                '{name="ACCOUNT_ID", value="__ACCOUNT_ID__"}, ' +
                '{name="ACCOUNT", value="__ACCOUNT__"}, ' +
                '{name="ENVIRONMENT", value="__ENV__"}, ' +
                '{name="spring.profiles.active", value="aws"}, ' +
                '{name="spring.jpa.generate-ddl", value="false"}, ' +
                '{name="spring.jpa.active", value="__ENV__"}, ' +
                '{name="env.accountId", value="__ACCOUNT_ID__"}, ' +
                '{name="env.accountType", value="__ACCOUNT__"}, ' +
                '{name="env.name", value="__ENV__"}, ' +
                '{name="spring.datasource.db", value="master"}]'
    }

    agent { label 'docker' }

    options {
        disableConcurrentBuilds()
        timestamps()
        ansiColor('xterm')
    }

    stages {


        stage("Test") {
            when {
                anyOf {
                    environment name: 'IMAGE_EXISTS', value: 'false'
                    branch 'dev'
                    branch 'master'
                }
            }
            agent {
                docker {
                    image "maven:${MAVEN_VERSION}"
                    args "-v /var/run/docker.sock:/var/run/docker.sock -u root"
                }
            }
            steps {
                sh "mvn -s settings.xml clean install"
                stash includes: "application/target/**", name: "target"
            }
        }

        stage('Is dev or tag?') {
            when {
                anyOf {
                    branch "dev"
                    tag pattern: /^\d+\.\d+\.\d$/, comparator: "REGEXP"
                }
            }
            steps {
                script {
                    DEPLOYMENT_ENV = env.BRANCH_NAME == "dev" ? PHOTON : PROTON
                }
            }
        }

        stage("Should deploy?") {
            when {
                expression { DEPLOYMENT_ENV == NOWHERE }
            }
            steps {
                script {
                    try {
                        timeout(time: 2, unit: 'MINUTES') {
                            choices = [PHOTON, GRAVITON, HADRON, MUON, ELECTRON]

                            if (env.BRANCH_NAME == "master" || env.BRANCH_NAME.startsWith("release/") || env.BRANCH_NAME.startsWith("hotfix/")) {
                                choices.add(NEUTRON)
                            }

                            DEPLOYMENT_ENV = input message: 'Where do you want to deploy?', ok: 'Deploy!',
                                    parameters: [choice(
                                            name: 'DEPLOYMENT_ENV',
                                            choices: choices,
                                            defaultValue: 'NOWHERE',
                                            description: 'What is the deployment environment?')]
                        }
                    } catch (err) {
                        echo("Either timeout or abort: ${err}")
                    }
                }
            }
        }

        stage("Change Control") {
            options {
                timeout(time: 10, unit: 'MINUTES')
            }
            when {
                expression { DEPLOYMENT_ENV == PROTON }
            }
            steps {
                script {
                    changeControl = input message: 'Please Enter a Change Control Number',
                            parameters: [string(defaultValue: '',
                                    description: '',
                                    name: 'Change Control Number')]

                    if (changeControl == '') {
                        error "You need to provide a valid change control number"
                    }
                    echo("Change Control Number: ${changeControl}")
                }
            }
        }
    }
}

def get_aws_details() {
    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                      credentialsId: "aws-paris-terraform-" + ACCOUNT_VALUE , secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
        script {
            tmp_deployment = "${DEPLOYMENT_ENV}".toLowerCase()
            return sh(script: 'aws ecs describe-tasks --cluster ' + tmp_deployment + ' --region eu-west-1 --tasks ' +
                    '$(aws ecs list-tasks --cluster ' + tmp_deployment + ' --region eu-west-1 --query "taskArns[*]" --output text) ' +
                    '--query "tasks[*].containers[*].[lastStatus,image]" --output text', returnStdout: true).replaceAll(
                    "097002022524.dkr.ecr.eu-west-1.amazonaws.com/", "").split("\n").sort().join("\n")
        }
    }
}

void deploy() {
    build job: '/paris/infrastructure-ecs-deploy', parameters: [ \
        [$class: 'StringParameterValue', name: 'aws_account_name', value: "${ACCOUNT}"], \
        [$class: 'StringParameterValue', name: 'env', value: "${ENV}"], \
        [$class: 'StringParameterValue', name: 'app_name', value: "${PROJECT}"], \
        [$class: 'StringParameterValue', name: 'app_path', value: "${APP_PATH}"], \
        [$class: 'StringParameterValue', name: 'ecr_string', value: "${DOCKER_IMAGE_NAME}"], \
        [$class: 'StringParameterValue', name: 'health_check_endpoint', value: "${HEALTH_CHECK_ENDPOINT}"], \
        [$class: 'StringParameterValue', name: 'app_sha', value: "${GIT_COMMIT}"], \
        [$class: 'StringParameterValue', name: 'tag_name', value: "${GIT_TAG}"], \
        [$class: 'StringParameterValue', name: 'fargate_cpu', value: "${FARGATE_CPU}"], \
        [$class: 'StringParameterValue', name: 'fargate_memory', value: "${FARGATE_MEMORY}"], \
        [$class: 'StringParameterValue', name: 'secrets_vars', value: "${SECRETS_VARS}"], \
        [$class: 'StringParameterValue', name: 'env_vars', value: "${ENV_VARS}"]
    ]
}

def slackSuccess(channel, message) {
    slackMessage(channel, message, "good")
}

def slackFailure(channel, message) {
    slackMessage(channel, message, "danger")
}

def slackMessage(channel, message, color) {
    slackSend(
            teamDomain: env.SLACK_TEAM_DOMAIN,
            token: env.SLACK_TOKEN,
            channel: channel,
            color: color,
            message: message
    )
}