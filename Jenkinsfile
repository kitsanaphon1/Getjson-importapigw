pipeline {
    agent any

    environment {
        // Source: ดึงจาก FastAPI ของคุณ
        API_SERVER_URL = "http://172.188.16.48:8000/openapi.json"
        
        // Target: IBM API Gateway (ตรวจสอบ Port ให้ชัวร์ ปกติ 5555 คือ Management Port)
        APIGW_URL      = "http://20.198.251.142:5555" 
        
        // Credentials ID จาก Jenkins
        APIGW_AUTH     = credentials('apigw-admin-password') 
    }

    stages {
        stage('Check Connectivity') {
            steps {
                echo "Checking connection to API Server and Gateway..."
                sh "curl -I ${API_SERVER_URL}"
                sh "curl -I ${APIGW_URL}/rest/apigateway/health" // เช็คว่า Gateway พร้อมรับแขกไหม
            }
        }

        stage('Extract OpenAPI Spec') {
            steps {
                echo "Downloading OpenAPI Spec..."
                sh "curl -s ${API_SERVER_URL} > swagger_spec.json"
                
                // ตรวจสอบความสมบูรณ์ของไฟล์
                sh "ls -lh swagger_spec.json"
                sh "grep 'openapi' swagger_spec.json"
            }
        }

        stage('Push to IBM API Gateway (POST)') {
            steps {
                echo "Importing API via REST API POST..."
                /* ใช้ API ของ webMethods เพื่อ Import:
                   -u คือ Username:Password
                   -d @swagger_spec.json คือการส่งไฟล์ไปใน Body
                */
                sh """
                curl -X POST "${APIGW_URL}/rest/apigateway/apis" \
                    -u "${APIGW_AUTH_USR}:${APIGW_AUTH_PSW}" \
                    -H "Content-Type: application/json" \
                    -d @swagger_spec.json
                """
            }
        }
    }

    post {
        success { echo 'Done! API updated via REST API.' }
        failure { echo 'Failed! Check network or API Gateway Logs.' }
        always {
            sh "rm -f swagger_spec.json || true"
        }
    }
}