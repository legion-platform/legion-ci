import java.text.SimpleDateFormat

class Globals {
    static String dockerTag = null
    static String dockerCacheArg = null
}

def legion

pipeline {
    agent any
    options{
        buildDiscarder(logRotator(numToKeepStr: '5'))
        disableConcurrentBuilds()
    }
    environment {
        buildWorkspace = "${WORKSPACE}"
        dexDockerimage = "k8s-dex"
        sharedLibPath = "deploy/legionPipeline.groovy"
        //parameters section
        //git branch to checkout 
        //Set main branch for Job SCM GitBranch
        //git branch to checkout legion
        gitBranchLegion = "${params.GitBranchLegion}"
        //git branch to checkout dex dockerfile
        gitBranchDex = "${params.GitBranchDex}"
        //Git repo - main organization repo
        mainGitRepo = "${params.MainGitRepo}"
        //Enable slack notifications bolean parameter
        slackNotifications = "${params.EnableSlackNotifications}"
        // Docker registry parameter
        dockerRegistry = "${params.DockerRegistry}"
        //Enable docker cache parameter
        enableDockerCache = "${params.EnableDockerCache}"
        //Get dex version from parameter
        dexVersion = "${params.DexVersion}"
        // Get version from parameter
        legionVersion = "${params.LegionVersion}"
    }
    stages {
        stage('Checkout and set build vars') {
            steps {
                dir ("${buildWorkspace}/dex") {
                    checkout([$class: 'GitSCM', branches: [[name: "${gitBranchDex}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: "${mainGitRepo}/dex.git"]]])
                }
                dir ("${buildWorkspace}/legion") {
                    checkout([$class: 'GitSCM', branches: [[name: "${gitBranchLegion}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: "${mainGitRepo}/legion.git"]]])
                    script{
                        Globals.dockerCacheArg = "${enableDockerCache}" ? '' : '--no-cache'
                        print("Check code for security issues")
                        sh "bash install-git-secrets-hook.sh install_hooks && git secrets --scan -r"
                        Globals.dockerTag = "${dexVersion}-legion-${legionVersion}"
                    }
                }
            }
        }
        stage('Build docker image Dex') {
            steps {
                dir ("${buildWorkspace}/dex") {
                    sh "docker build ${Globals.dockerCacheArg} -t ${dockerRegistry}/${dexDockerimage}:${BUILD_NUMBER} -f Dockerfile ."
                }
            }
        }
        stage('Push docker image Dex') {
            steps {
                sh """
                docker tag ${dockerRegistry}/${dexDockerimage}:${BUILD_NUMBER} ${dockerRegistry}/${dexDockerimage}:latest
                docker tag ${dockerRegistry}/${dexDockerimage}:${BUILD_NUMBER} ${dockerRegistry}/${dexDockerimage}:${Globals.dockerTag}
                docker push ${dockerRegistry}/${dexDockerimage}:${Globals.dockerTag}
                docker push ${dockerRegistry}/${dexDockerimage}:latest
                """
            }
        }
    }
    post {
        always {
            script {
                legion = load "legion/${sharedLibPath}"
                legion.notifyBuild(currentBuild.currentResult)
            }
            deleteDir()
        }
    }
}