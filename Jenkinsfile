#!groovy


node {
    // 基础环境变量映射
    def SF_USERNAME = env.SF_USERNAME
    def SF_CLIENT_ID = env.SF_CLIENT_ID
    def TEST_LEVEL = 'RunLocalTests'
    def PACKAGE_NAME = 'dreamhousetest' 
    def PACKAGE_VERSION
    def SF_INSTANCE_URL = "https://login.salesforce.com"

    // -------------------------------------------------------------------------
    // 检出代码
    // -------------------------------------------------------------------------
    stage('checkout source') {
        checkout scm
    }

    // -------------------------------------------------------------------------
    // 使用 Salesforce 凭据进行操作
    // -------------------------------------------------------------------------
    withEnv(["HOME=${env.WORKSPACE}"]) {
        withCredentials([file(credentialsId: 'sf-jwt-key', variable: 'server_key_file')]) {

            // 1. 授权 Dev Hub (必须要保留，因为打包需要 Dev Hub 权限)
            stage('Authorize DevHub') {
                def rc = command "sf org login jwt --instance-url ${SF_INSTANCE_URL} --client-id ${SF_CLIENT_ID} --username ${SF_USERNAME} --jwt-key-file ${server_key_file} --set-default-dev-hub --alias DevHub"
                
                // 打印当前 Org 信息，确保连接的是那个有 2 个包的正确 Dev Hub
                command "sf org display --target-org DevHub"
                command "sf package list --target-dev-hub DevHub"

                if (rc != 0) {
                    error 'Salesforce dev hub org authorization failed.'
                }
            }

            // --- [已注释] 步骤 2, 3, 4：源码验证阶段需要 Scratch Org，暂时跳过 ---
            /*
            stage('Create Test Scratch Org') { ... }
            stage('Deploy and Test Source') { ... }
            stage('Delete Test Org') { ... }
            */

            // 2. 直接创建软件包版本 (不消耗 Scratch Org 配额)
            // 1. 删除脚本顶部的 import groovy.json.JsonSlurperClassic

            // 2. 修改 Create Package Version 阶段的代码
            stage('Create Package Version') {
                def createCmd = "sf package version create --package ${PACKAGE_NAME} --installation-key-bypass --code-coverage --wait 20 --json --target-dev-hub DevHub"
                
                def output
                if (isUnix()) {
                    output = sh(returnStdout: true, script: createCmd)
                } else {
                    output = bat(returnStdout: true, script: createCmd).trim()
                    output = output.readLines().find { it.contains('{') && it.contains('}') } ?: output
                }

                // 使用 Jenkins 内置的 readJSON 替代 JsonSlurper
                def response = readJSON text: output

                if (response.status == 0) {
                    PACKAGE_VERSION = response.result.SubscriberPackageVersionId
                    echo "Successfully created package version: ${PACKAGE_VERSION}"
                } else {
                    error "Package version creation failed: ${response.message}"
                }
            }

            // --- [已注释] 步骤 6, 7, 8：验证安装阶段需要 Scratch Org，暂时跳过 ---
            /*
            stage('Create Install Verification Org') { ... }
            stage('Install and Verify Package') { ... }
            stage('Final Cleanup') { ... }
            */
        }
    }
}

// 通用命令处理函数 (增加了 def 以优化内存)
def command(script) {
    if (isUnix()) {
        return sh(returnStatus: true, script: script);
    } else {
        return bat(returnStatus: true, script: script);
    }
}