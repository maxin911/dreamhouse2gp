#!groovy

node {
    // 1. 基础变量配置
    def SF_USERNAME = env.SF_USERNAME
    def SF_CLIENT_ID = env.SF_CLIENT_ID
    def PACKAGE_NAME = 'dreamhousetest' // 与 sfdx-project.json 中的 Alias 保持一致
    def SF_INSTANCE_URL = "https://login.salesforce.com"
    def PACKAGE_VERSION = ""

    stage('Checkout') {
        checkout scm
    }

    withEnv(["HOME=${env.WORKSPACE}"]) {
        withCredentials([file(credentialsId: 'sf-jwt-key', variable: 'server_key_file')]) {

            // 2. 授权阶段
            stage('Authorize DevHub') {
                def rc = command "sf org login jwt --instance-url ${SF_INSTANCE_URL} --client-id ${SF_CLIENT_ID} --username ${SF_USERNAME} --jwt-key-file ${server_key_file} --set-default-dev-hub --alias DevHub"
                
                // 调试：确保登录的是正确的新 Dev Hub
                command "sf org display --target-org DevHub"
                
                if (rc != 0) error 'Salesforce DevHub 授权失败，请检查 Credentials 和 Client ID。'
            }

            // 3. 创建软件包版本 (跳过 Scratch Org 以节省配额)
            stage('Create Package Version') {
                echo "正在创建软件包版本，请耐心等待 (约 5-15 分钟)..."
                
                // 注意：这里去掉了 --json，因为我们要用正则从标准输出捕获，这样更不容易出错
                def createCmd = "sf package version create --package ${PACKAGE_NAME} --installation-key-bypass --code-coverage --wait 20 --target-dev-hub DevHub"
                
                def output = ""
                if (isUnix()) {
                    output = sh(returnStdout: true, script: createCmd).trim()
                } else {
                    output = bat(returnStdout: true, script: createCmd).trim()
                }
                
                echo "--- CLI Output Start ---"
                echo output
                echo "--- CLI Output End ---"

                // 使用正则表达式提取 04t 开头的 ID
                // 匹配格式如：Successfully created the package version [04t8000000XXXXX]
                def matcher = output =~ /04t[a-zA-Z0-9]{15,18}/
                if (matcher.find()) {
                    PACKAGE_VERSION = matcher.group(0)
                    echo "成功提取版本 ID: ${PACKAGE_VERSION}"
                } else {
                    error "未能从输出中找到 04t ID。请检查上方 CLI 输出是否包含错误信息。"
                }
            }

            // 4. 下一步提示
            stage('Summary') {
                echo "====================================================="
                echo "构建成功！"
                echo "Package Name: ${PACKAGE_NAME}"
                echo "Package Version ID: ${PACKAGE_VERSION}"
                echo "安装命令: sf package install --package ${PACKAGE_VERSION} --target-org <YOUR_ORG>"
                echo "====================================================="
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