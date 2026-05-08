#!groovy

node {
    // 1. 基础变量配置
    def SF_USERNAME = env.SF_USERNAME
    def SF_CLIENT_ID = env.SF_CLIENT_ID
    def PACKAGE_NAME = 'dreamhousetest' // 与 sfdx-project.json 中的 Alias 保持一致
    def SF_INSTANCE_URL = "https://login.salesforce.com"
    def TEST_LEVEL = 'RunLocalTests'
    def PACKAGE_VERSION = ""

    stage('Checkout') {
        checkout scm
    }

    withEnv(["HOME=${env.WORKSPACE}"]) {
        withCredentials([file(credentialsId: 'sf-jwt-key', variable: 'server_key_file')]) {

            // 2. 授权阶段
            stage('Authorize DevHub') {
                def rc = command "sf org login jwt --instance-url ${SF_INSTANCE_URL} --client-id ${SF_CLIENT_ID} --username ${SF_USERNAME} --jwt-key-file ${server_key_file} --set-default-dev-hub --alias DevHub"
                
                // 打印信息确保连接正确
                command "sf org display --target-org DevHub"
                
                if (rc != 0) error 'Salesforce DevHub 授权失败。'
            }

            // 3. 创建临时组织进行源码验证 (如配额耗尽可跳过)
            stage('Source Verification (Scratch Org)') {
                echo "正在创建 Scratch Org 并验证源代码..."
                def rc = command "sf org create scratch --target-dev-hub DevHub --definition-file config/project-scratch-def.json --alias ciorg --wait 10 --duration-days 1 --set-default"
                if (rc == 0) {
                    try {
                        command "sf project deploy start --target-org ciorg --wait 10"
                        def testRc = command "sf apex run test --target-org ciorg --wait 10 --result-format tap --code-coverage --test-level ${TEST_LEVEL}"
                        if (testRc != 0) error '源代码单元测试未通过。'
                    } finally {
                        command "sf org delete scratch --target-org ciorg --no-prompt"
                    }
                } else {
                    echo "警告: Scratch Org 创建失败 (可能达到配额上限)，跳过源码验证，直接尝试打包。"
                }
            }

            // 4. 创建软件包版本 (使用您验证成功的正则逻辑)
            stage('Create Package Version') {
                echo "正在生成 2GP 软件包版本..."
                def createCmd = "sf package version create --package ${PACKAGE_NAME} --installation-key-bypass --code-coverage --wait 20 --target-dev-hub DevHub"
                
                def output = isUnix() ? sh(returnStdout: true, script: createCmd).trim() : bat(returnStdout: true, script: createCmd).trim()
                
                echo output

                def matcher = output =~ /04t[a-zA-Z0-9]{15,18}/
                if (matcher.find()) {
                    PACKAGE_VERSION = matcher.group(0)
                    echo "成功获取 Package Version ID: ${PACKAGE_VERSION}"
                } else {
                    error "未能捕获 04t ID，打包失败。"
                }
                
                echo "等待 5 分钟确保包同步至全球节点..."
                sleep 300
            }

            // 5. 安装验证 (在新的 Scratch Org 中测试包安装)
            stage('Install Verification') {
                echo "验证安装包 ${PACKAGE_VERSION}..."
                def rc = command "sf org create scratch --target-dev-hub DevHub --definition-file config/project-scratch-def.json --alias installorg --wait 10 --duration-days 1"
                if (rc == 0) {
                    try {
                        def installRc = command "sf package install --package ${PACKAGE_VERSION} --target-org installorg --wait 15 --no-prompt"
                        if (installRc != 0) error '软件包安装验证失败。'
                        echo "安装验证成功。"
                    } finally {
                        command "sf org delete scratch --target-org installorg --no-prompt"
                    }
                } else {
                    echo "配额限制，跳过安装验证阶段。"
                }
            }

            // 6. 推广版本 (Promote)
            stage('Promote Package') {
                // 这是一个安全开关：只有点击确认才会执行 Promote
                input message: "是否确认将版本 ${PACKAGE_VERSION} 推广为正式版(Released)？注意：此操作不可逆。"
                
                echo "正在 Promote 软件包..."
                def rc = command "sf package version promote --package ${PACKAGE_VERSION} --no-prompt --target-dev-hub DevHub"
                if (rc != 0) {
                    error "Promote 失败。请确保代码覆盖率达到 75% 以上。"
                } else {
                    echo "恭喜！版本 ${PACKAGE_VERSION} 现在已是正式版。"
                }
            }

            // 7. 总结
            stage('Summary') {
                echo "====================================================="
                echo "Pipeline 执行完毕"
                echo "最终正式版 ID: ${PACKAGE_VERSION}"
                echo "====================================================="
            }
        }
    }
}

def command(script) {
    if (isUnix()) {
        return sh(returnStatus: true, script: script);
    } else {
        return bat(returnStatus: true, script: script);
    }
}