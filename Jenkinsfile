pipeline {
    agent any
    environment {
        ENV_PROFILE = ''
        TAG = "${currentBuild.timeInMillis}"
        MAVEN_HOME = ''
        BASE_DIR = ''
        DEPLOY_DIR = ''
    }
    stages {
        stage('检查参数'){
            steps{
                ansiColor('xterm'){
                    script{
                        if(params.SERVER_LIST == ""){
                            error("\u001B[31m请选择要部署的服务器IP:SERVER_LIST!\u001B[0m\u274C")
                        }
                        else if(params.ACTION == ""){
                            error("\u001B[31m请选择要操作的Action!\u001B[0m\u274C")
                        }
                        else if(params.SERVICE_NAME == ""){
                            error("\u001B[31m请选择要bu'shu部署的服务名称SERVICE_NAME!\u001B[0m\u274C")
                        }
                        else if(params.Release_Comment == ""){
                            error("\u001B[31m请选择此次操作的描述说明:Release_Comment!\u001B[0m\u274C")
                        }
                    }
                }
            }
        }
        stage('打包编译\u273F'){
            when { environment name: 'ACTION', value: 'Deploy' }
            steps {
                ansiColor('xterm'){
                    timestamps{
                        echo "\u27A1开始拉取gitlab代码......"
                        git branch: 'master',
                        credentialsId: '<credentialsId>',
                        url: '<url>'

                        echo "\u27A1开始打包编译:......"
                        sh "${MAVEN_HOME}/bin/mvn clean package -U -DskipTests -P${ENV_PROFILE}"
                        echo "\u001B[34m编译完成\u2713\u001B[0m"
                    }
                }
            }
        }

        stage('Jar包分发\u273F'){
            when { environment name: 'ACTION', value: 'Deploy' }
            steps{
                ansiColor('xterm'){
                    timestamps{
                        script{
                            def labels = SERVER_LIST.split(',')
                            def builders = [:]
                            for (la in labels) {
                                def label = la
                                builders[label] = {
                                    sh "rsync -av ${WORKSPACE}/${SERVICE_NAME}/target/${SERVICE_NAME}.jar <target_dir>"
                                    echo "\u001B[34m分发完成\u2713\u001B[0m"
                                }
                            }
                            parallel builders
                       }
                    }
                }
           }
        }

        stage('部署\u273F'){
            steps{
                ansiColor('xterm'){
                    timestamps{
                        withEnv(['JENKINS_NODE_COOKIE=dontkillme']) {
                            script {
                                def labels = SERVER_LIST.split(',')
                                def builders = [:]
                                for (la in labels) {
                                    def label = la
                                    node(label) {
                                        sh'''
                                            case "$ACTION" in
                                                "Deploy")
                                                    sh ${BASE_DIR}/server.sh stop ${SERVICE_NAME}
                                                    /bin/cp ${DEPLOY_DIR}/${SERVICE_NAME}/build/${SERVICE_NAME}.jar ${DEPLOY_DIR}/${SERVICE_NAME}/
                                                    sh ${BASE_DIR}/server.sh start ${SERVICE_NAME}
                                                    ;;
                                                "Rollback")
                                                    sh ${BASE_DIR}/server.sh stop ${SERVICE_NAME}
                                                    /bin/cp ${DEPLOY_DIR}/${SERVICE_NAME}/build/${ROLLBACK_VERSION}/${SERVICE_NAME}.jar ${DEPLOY_DIR}/${SERVICE_NAME}/
                                                    sh ${BASE_DIR}/server.sh start ${SERVICE_NAME}
                                                    ;;
                                                *)
                                                    ;;
                                            esac
                                        '''
                                        echo "部署完成: $label"
                                        sleep 10
                                  }

                                }

                                // echo "\u001B[34m${SERVICE_NAME}部署完成\u2713\u001B[0m"
                            }
                        }
                    }
                }
            }
        }

        stage('备份\u273F'){
            when { environment name: 'ACTION', value: 'Deploy' }
            steps{
                ansiColor('xterm'){
                    timestamps{
                        script{
                            def labels = SERVER_LIST.split(',')
                            def builders = [:]
                            for (la in labels) {
                                def label = la
                                builders[label] = {
                                    node(label) {
                                        sh '''
                                            echo "\u27A1清理备份..."
                                            cd ${DEPLOY_DIR}/${SERVICE_NAME}/build
                                            ls -tF |grep '/$'|awk '{if (NR > 3) print $0 }'|xargs rm -rf
                                            echo "\u001B[34m备份清理完成\u2713\u001B[0m"
                                            nowtime=`date "+%Y%m%d%H%M%S" -d @${TAG:0:10}`
                                            mkdir -p ${DEPLOY_DIR}/${SERVICE_NAME}/build/$nowtime
                                            /bin/cp ${DEPLOY_DIR}/${SERVICE_NAME}/build/${SERVICE_NAME}.jar ${DEPLOY_DIR}/${SERVICE_NAME}/build/$nowtime/
                                        '''
                                    }
                                }
                            }
                            parallel builders
                       }
                        echo "\u001B[34m备份完成\u2713\u001B[0m"
                    }
                }
            }
        }

        stage('Post Build\u273F'){
            agent {
                label "master"
            }
            steps{
                timestamps{
                    script {
                        wrap([$class: 'BuildUser']) {
                            manager.addShortText("${BUILD_USER}")
                        }
                        def s_list = SERVER_LIST.split(',')
                        for (int i = 0; i < s_list.size(); ++i) {
                            manager.addShortText(${s_list.split('.')[-1]})
                        }
                        manager.addShortText(env.SERVICE_NAME)
                        manager.addShortText(env.SERVER_LIST)
                        if (env.ACTION == "Rollback"){
                            manager.addShortText(env.ROLLBACK_VERSION)
                        }
                        else if (env.ACTION == "Deploy"){
                            def cr = new Date("${TAG}" as long).format("yyyyMMddHHmmss")
                            manager.addShortText(cr)
                        }
                        manager.addShortText(manager.envVars['ACTION'])
                    }
                }
            }
        }
    }
}