#!groovy
import groovy.json.JsonSlurperClassic

node {
    // 基础环境变量映射（建议在 Jenkins Job 界面配置这些环境变量）
    def SF_USERNAME = env.SF_USERNAME
    def SF_CLIENT_ID = env.SF_CLIENT_ID
    def TEST_LEVEL = 'RunLocalTests'
    def PACKAGE_NAME = 'dreamhousetest' // 你的包 ID
    def PACKAGE_VERSION
    def SF_INSTANCE_URL = env.SF_INSTANCE_URL ?: "https://test.salesforce.com"

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
        // 这里的 'sf-jwt-key' 必须与你在 Jenkins Credentials 存储的 ID 一致
        withCredentials([file(credentialsId: 'sf-jwt-key', variable: 'server_key_file')]) {

            // 1. 授权 Dev Hub
            stage('Authorize DevHub') {
                // 直接使用 sf，不再依赖 toolbelt 变量
                rc = command "sf org login jwt --instance-url ${SF_INSTANCE_URL} --client-id ${SF_CLIENT_ID} --username ${SF_USERNAME} --jwt-key-file ${server_key_file} --set-default-dev-hub --alias DevHub"
                if (rc != 0) {
                    error 'Salesforce dev hub org authorization failed.'
                }
            }

            // 2. 创建第一个临时组织 (用于源代码验证)
            stage('Create Test Scratch Org') {
                rc = command "sf org create scratch --target-dev-hub DevHub --set-default --definition-file config/project-scratch-def.json --alias ciorg --wait 10 --duration-days 1"
                if (rc != 0) {
                    error 'Salesforce test scratch org creation failed.'
                }
            }

            // 3. 推送代码并运行测试 (确保源代码健康)
            stage('Deploy and Test Source') {
                rc = command "sf project deploy start --target-org ciorg --wait 10"
                if (rc != 0) error 'Source deployment failed.'
                
                rc = command "sf apex run test --target-org ciorg --wait 10 --result-format tap --code-coverage --test-level ${TEST_LEVEL}"
                if (rc != 0) error 'Unit tests failed in source org.'
            }

            // 4. 清理第一个临时组织
            stage('Delete Test Org') {
                command "sf org delete scratch --target-org ciorg --no-prompt"
            }

            // 5. 创建软件包版本 (添加了 --code-coverage)
            stage('Create Package Version') {
                // 关键点：添加了 --code-coverage 以便后续可以 Promote
                def createCmd = "sf package version create --package ${PACKAGE_NAME} --installation-key-bypass --code-coverage --wait 20 --json --target-dev-hub DevHub"
                
                if (isUnix()) {
                    output = sh(returnStdout: true, script: createCmd)
                } else {
                    output = bat(returnStdout: true, script: createCmd).trim()
                    output = output.readLines().drop(1).join(" ")
                }

                def jsonSlurper = new JsonSlurperClassic()
                def response = jsonSlurper.parseText(output)

                // 获取生成的 04t ID
                PACKAGE_VERSION = response.result.SubscriberPackageVersionId
                echo "Successfully created package version: ${PACKAGE_VERSION}"
                
                // 等待同步，避免立刻安装时报找不到包的错误
                echo "Waiting 5 minutes for package replication..."
                sleep 300
            }

            // 6. 创建第二个临时组织 (用于验证安装包)
            stage('Create Install Verification Org') {
                rc = command "sf org create scratch --target-dev-hub DevHub --definition-file config/project-scratch-def.json --alias installorg --wait 10 --duration-days 1"
                if (rc != 0) error 'Install scratch org creation failed.'
            }

            // 7. 安装并验证包
            stage('Install and Verify Package') {
                rc = command "sf package install --package ${PACKAGE_VERSION} --target-org installorg --wait 15 --no-prompt"
                if (rc != 0) error 'Package installation failed.'
                
                // 再次运行测试，确保包内逻辑正确
                rc = command "sf apex run test --target-org installorg --result-format tap --code-coverage --test-level ${TEST_LEVEL} --wait 10"
                if (rc != 0) error 'Tests failed in installed package.'
            }

            // 8. 最终清理
            stage('Final Cleanup') {
                command "sf org delete scratch --target-org installorg --no-prompt"
            }
        }
    }
}

// 通用命令处理函数
def command(script) {
    if (isUnix()) {
        return sh(returnStatus: true, script: script);
    } else {
        return bat(returnStatus: true, script: script);
    }
}