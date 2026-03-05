pipeline {
    agent any

    environment {
        // ข้อมูล API Server (Source)
        API_SERVER_URL = "http://172.188.16.48:8000/openapi.json"
        
        // ข้อมูล IBM API Gateway (Target)
        APIGW_URL      = "http://20.198.251.142:5555" 
        
        // ข้อมูล API ที่จะสร้างบน Gateway
        API_NAME       = "customer-erp-API"
        API_VERSION    = "1.0.${BUILD_NUMBER}"
        
        // เรียกใช้ ID ของ Credentials ที่คุณสร้างใน Jenkins
        // โดยระบบจะแยกเป็นตัวแปร APIGW_USR และ APIGW_PSW ให้โดยอัตโนมัติ
        APIGW_AUTH     = credentials('apigw-admin-password') 
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
                
                // ตรวจสอบว่าไฟล์มีเนื้อหาและเป็นรูปแบบ OpenAPI จริง
                sh "ls -lh swagger_spec.json"
                sh "grep 'openapi' swagger_spec.json || (echo 'Error: Invalid OpenAPI file' && exit 1)"
            }
        }

        stage('Push to IBM API Gateway') {
            steps {
                echo "Importing API to IBM webMethods API Gateway..."
                // ใช้ตัวแปรที่ได้จาก credentials() 
                // ${APIGW_AUTH_USR} จะได้ Username และ ${APIGW_AUTH_PSW} จะได้ Password
                sh """
                apigw-cli import -f swagger_spec.json \
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
        failure { echo 'Failed! Please check apigw-cli logs or network connectivity.' }
        always {
            sh "rm -f swagger_spec.json"
        }
    }
}