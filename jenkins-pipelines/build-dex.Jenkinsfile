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
        build_workspace = "${WORKSPACE}"
        def dex_dockerimage = "k8s-dex"
        shared_lib_path = "deploy/legionPipeline.groovy"
        //parameters section
        //git branch to checkout 
        //Set main branch for Job SCM GitBranch
        //git branch to checkout legion
        git_branch_legion = "${params.GitBranchLegion}"
        //git branch to checkout dex dockerfile
        git_branch_dex = "${params.GitBranchDex}"
        //Git repo - main organization repo
        main_git_repo = "${params.MainGitRepo}"
        //Enable slack notifications bolean parameter
        slack_notifications = "${params.EnableSlackNotifications}"
        // Docker registry parameter
        docker_registry = "${params.DockerRegistry}"
        //Enable docker cache parameter
        enable_docker_cache = "${params.EnableDockerCache}"
        //Get dex version from parameter
        dex_version = "${params.DexVersion}"
        // Get version from parameter
        legion_version = "${params.LegionVersion}"
    }
    stages {
        stage('Checkout and set build vars') {
            steps {
                dir ("${build_workspace}/dex") {
                    checkout([$class: 'GitSCM', branches: [[name: "${git_branch_dex}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: "${main_git_repo}/dex.git"]]])
                }
                dir ("${build_workspace}/legion") {
                    checkout([$class: 'GitSCM', branches: [[name: "${git_branch_legion}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: "${main_git_repo}/legion.git"]]])
                    script{
                        Globals.dockerCacheArg = ${enable_docker_cache} ? '' : '--no-cache'
                        print("Check code for security issues")
                        sh "bash install-git-secrets-hook.sh install_hooks && git secrets --scan -r"
                        Globals.dockerTag = "${dex_version}-legion-${legion_version}"
                    }
                }
            }
        }
        stage('Build docker image Dex') {
            steps {
                dir ("${build_workspace}/dex") {
                    sh "docker build ${Globals.dockerCacheArg} -t ${docker_registry}/${dex_dockerimage}:${BUILD_NUMBER} -f Dockerfile ."
                }
            }
        }
        stage('Push docker image Dex') {
            steps {
                sh """
                docker tag ${docker_registry}/${dex_dockerimage}:${BUILD_NUMBER} ${docker_registry}/${dex_dockerimage}:latest
                docker tag ${docker_registry}/${dex_dockerimage}:${BUILD_NUMBER} ${docker_registry}/${dex_dockerimage}:${Globals.dockerTag}
                docker push ${docker_registry}/${dex_dockerimage}:${Globals.dockerTag}
                docker push ${docker_registry}/${dex_dockerimage}:latest
                """
            }
        }
    }
    post {
      always {
          script {
              legion = load "legion/${shared_lib_path}"
              legion.notifyBuild(currentBuild.currentResult)
          }
          deleteDir()
      }
  }
}