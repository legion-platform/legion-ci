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
        /// Input parameters
        //git branch to checkout 
        //Set main branch for Job SCM GitBranch
        //git branch to checkout legion
        param_gitBranchLegion = "${params.GitBranchLegion}"
        //git branch to checkout dex dockerfile
        param_gitBranchDex = "${params.GitBranchDex}"
        //Git repo - main organization repo
        param_mainGitRepo = "${params.MainGitRepo}"
        //Enable slack notifications bolean parameter
        param_slackNotifications = "${params.EnableSlackNotifications}"
        // Docker registry parameter
        param_dockerRegistry = "${params.DockerRegistry}"
        //Enable docker cache parameter
        param_enableDockerCache = "${params.EnableDockerCache}"
        //Get dex version from parameter
        param_dexVersion = "${params.DexVersion}"
        // Get version from parameter
        param_legionVersion = "${params.LegionVersion}"
        ///Job parameters
        buildWorkspace = "${WORKSPACE}"
        dexDockerimage = "k8s-dex"
        sharedLibPath = "deploy/legionPipeline.groovy"
        
    }
    stages {
        stage('Checkout and set build vars') {
            steps {
                dir ("${buildWorkspace}/dex") {
                    checkout([$class: 'GitSCM', branches: [[name: "${param_gitBranchDex}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: "${param_mainGitRepo}/dex.git"]]])
                }
                dir ("${buildWorkspace}/legion") {
                    checkout([$class: 'GitSCM', branches: [[name: "${param_gitBranchLegion}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: "${param_mainGitRepo}/legion.git"]]])
                    script{
                        Globals.dockerCacheArg = "${param_enableDockerCache}" ? '' : '--no-cache'
                        print("Check code for security issues")
                        sh "bash install-git-secrets-hook.sh install_hooks && git secrets --scan -r"
                        Globals.dockerTag = "${param_dexVersion}-legion-${param_legionVersion}"
                    }
                }
            }
        }
        stage('Build docker image Dex') {
            steps {
                dir ("${buildWorkspace}/dex") {
                    sh "docker build ${Globals.dockerCacheArg} -t ${param_dockerRegistry}/${dexDockerimage}:${BUILD_NUMBER} -f Dockerfile ."
                }
            }
        }
        stage('Push docker image Dex') {
            steps {
                sh """
                docker tag ${param_dockerRegistry}/${dexDockerimage}:${BUILD_NUMBER} ${param_dockerRegistry}/${dexDockerimage}:latest
                docker tag ${param_dockerRegistry}/${dexDockerimage}:${BUILD_NUMBER} ${param_dockerRegistry}/${dexDockerimage}:${Globals.dockerTag}
                docker push ${param_dockerRegistry}/${dexDockerimage}:${Globals.dockerTag}
                docker push ${param_dockerRegistry}/${dexDockerimage}:latest
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