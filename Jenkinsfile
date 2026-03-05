pipeline {
    agent any

    environment {
        // Source API Server
        API_SERVER_URL = "http://172.188.16.48:8000/openapi.json"
        
        // Target IBM API Gateway
        APIGW_URL      = "http://20.198.251.142:5555" 
        
        // API Info
        API_NAME       = "customer-erp-API"
        API_VERSION    = "1.0.${BUILD_NUMBER}"
        
        // 1. ตรวจสอบว่า ID 'apigw-admin-password' ตรงกับใน Jenkins Credentials หรือไม่
        APIGW_AUTH     = credentials('apigw-admin-password') 
        
        // 2. ระบุ Full Path ของ apigw-cli (แก้ให้ตรงกับที่วางไว้ในเครื่อง Jenkins)
        // เช่น /opt/SoftwareAG/common/lib/cli/bin/apigw-cli
        CLI_PATH       = "/usr/local/bin/apigw-cli" 
    }

    stages {
        stage('Check Connectivity') {
            steps {
                echo "Checking connection to API Server..."
                sh "curl -I ${API_SERVER_URL}"
            }
        }

        stage('Extract OpenAPI Spec') {
            steps {
                echo "Downloading OpenAPI Spec from FastAPI..."
                sh "curl -s ${API_SERVER_URL} > swagger_spec.json"
                
                sh "ls -lh swagger_spec.json"
                sh "grep 'openapi' swagger_spec.json || (echo 'Error: Invalid OpenAPI file' && exit 1)"
            }
        }

        stage('Push to IBM API Gateway') {
            steps {
                echo "Importing API to IBM webMethods API Gateway..."
                // ใช้ตัวแปรที่ Jenkins เจนให้คือ _USR และ _PSW
                sh """
                ${CLI_PATH} import -f swagger_spec.json \
                    -u ${APIGW_AUTH_USR} \
                    -p ${APIGW_AUTH_PSW} \
                    -url ${APIGW_URL} \
                    -name "${API_NAME}" \
                    -v "${API_VERSION}"
                """
            }
        }
    }

    post {
        success { echo 'Done! API has been updated on IBM Gateway.' }
        failure { echo 'Failed! Please check apigw-cli path or network connectivity.' }
        always {
            // ป้องกัน Error หากไฟล์ไม่มีอยู่จริง
            sh "rm -f swagger_spec.json || true"
        }
    }
}