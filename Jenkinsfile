pipeline {
    agent any

    environment {
        // Source API (FastAPI)
        API_SERVER_URL = "http://172.188.16.48:8000/openapi.json"
        
        // Target IBM API Gateway
        APIGW_URL      = "http://20.198.251.142:5555" 
        
        // ข้อมูล API ที่ต้องการให้ปรากฏบน Gateway
        API_NAME       = "customer-erp-API"
        API_VERSION    = "1.0.${BUILD_NUMBER}"

        // Credentials ID จาก Jenkins
        APIGW_AUTH     = credentials('apigw-admin-password') 
    }

    stages {
        stage('Check Connectivity') {
            steps {
                echo "Checking connection to API Server and Gateway..."
                sh "curl -I ${API_SERVER_URL}"
                // เพิ่ม -u เพื่อไม่ให้ติด 401 Access Denied ตอนเช็ค Health
                sh "curl -u ${APIGW_AUTH_USR}:${APIGW_AUTH_PSW} -I ${APIGW_URL}/rest/apigateway/health"
            }
        }

        stage('Extract OpenAPI Spec') {
            steps {
                echo "Downloading OpenAPI Spec..."
                sh "curl -s ${API_SERVER_URL} > swagger_spec.json"
                sh "ls -lh swagger_spec.json"
            }
        }

        stage('Push to IBM API Gateway') {
            steps {
                echo "Importing API via Multipart Form Data..."
                /* แก้ไขการใช้ curl:
                   -F "apiDefinition=@..." คือการระบุฟิลด์ที่ Gateway ต้องการ
                   -F "type=openapi" เพื่อบอกว่าเป็นไฟล์ประเภทไหน
                */
                sh """
                curl -X POST "${APIGW_URL}/rest/apigateway/apis" \
                    -u "${APIGW_AUTH_USR}:${APIGW_AUTH_PSW}" \
                    -H "Accept: application/json" \
                    -F "apiDefinition=@swagger_spec.json" \
                    -F "apiName=${API_NAME}" \
                    -F "apiVersion=${API_VERSION}" \
                    -F "type=openapi"
                """
            }
        }
    }

    post {
        success { echo 'Congratulations! API has been imported successfully.' }
        failure { echo 'Failed! Please check API Gateway logs for detail.' }
        always {
            sh "rm -f swagger_spec.json || true"
        }
    }
}