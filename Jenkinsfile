pipeline {
    agent any

    environment {
        API_SERVER_URL = "http://172.188.16.48:8000/openapi.json"
        APIGW_URL      = "http://20.198.251.142:5555" 
        API_NAME       = "customer-erp-API"
        API_VERSION    = "1.0.${BUILD_NUMBER}"
        APIGW_AUTH     = credentials('apigw-admin-password') 
    }

    stages {
        stage('Check Connectivity') {
            steps {
                echo "Checking connection to API Server and Gateway..."
                sh "curl -I ${API_SERVER_URL}"
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
                /* จุดที่แก้ไข:
                   1. เพิ่ม ;type=application/json หลังชื่อไฟล์เพื่อให้ Gateway ทราบชนิดไฟล์ชัดเจน
                   2. เพิ่มฟิลด์ apiType=REST เพื่อระบุประเภท API เหมือนที่เลือกในหน้าเว็บ
                */
                sh """
                curl -X POST "${APIGW_URL}/rest/apigateway/apis" \
                    -u "${APIGW_AUTH_USR}:${APIGW_AUTH_PSW}" \
                    -H "Accept: application/json" \
                    -F "apiDefinition=@swagger_spec.json;type=application/json" \
                    -F "apiName=${API_NAME}" \
                    -F "apiVersion=${API_VERSION}" \
                    -F "type=openapi" \
                    -F "apiType=REST"
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