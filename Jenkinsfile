pipeline {
    agent any

    environment {
        API_SERVER_URL = "http://172.188.16.48:8000/openapi.json"
        APIGW_URL      = "http://20.198.251.142:5555"
        API_NAME       = "customer-erp-API"
        // ใช้เลข Build ของ Jenkins เพื่อแยกเวอร์ชั่น (ตามแนวทาง Managing versions)
        API_VERSION    = "1.0.${BUILD_NUMBER}"
    }

    stages {
        stage('Check Connectivity') {
            steps {
                echo "Checking connection to API Server and Gateway..."
                withCredentials([usernamePassword(
                    credentialsId: 'apigw-admin-password',
                    usernameVariable: 'APIGW_AUTH_USR',
                    passwordVariable: 'APIGW_AUTH_PSW'
                )]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        curl -fsSI "$API_SERVER_URL" >/dev/null
                        curl -fsSI -u "$APIGW_AUTH_USR:$APIGW_AUTH_PSW" \
                          "$APIGW_URL/rest/apigateway/health" >/dev/null
                    '''
                }
            }
        }

        stage('Extract & Patch Spec') {
            steps {
                echo "Downloading and Patching OpenAPI Spec..."
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    curl -fsSL "$API_SERVER_URL" -o swagger_spec.json

                    # 1. ใช้ jq บังคับเวอร์ชั่นเป็น 3.0.0 เพื่อให้ API Gateway ยอมรับ (ถ้า FastAPI ส่งมาเป็น 3.1.0)
                    # 2. อัปเดตเลขเวอร์ชั่นในไฟล์ให้ตรงกับ Jenkins Build
                    jq '.openapi = "3.0.0" | .info.version = "'${API_VERSION}'"' swagger_spec.json > patched_spec.json
                    
                    mv patched_spec.json swagger_spec.json
                '''
            }
        }

        stage('Push to IBM API Gateway') {
            steps {
                echo "Importing API via REST API..."
                withCredentials([usernamePassword(
                    credentialsId: 'apigw-admin-password',
                    usernameVariable: 'APIGW_AUTH_USR',
                    passwordVariable: 'APIGW_AUTH_PSW'
                )]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail

                        # ส่งไฟล์แบบ Multipart Form Data
                        # ระบุ Content-Type ของไฟล์เป็น application/json
                        http_code=$(curl -sS -o resp.json -w "%{http_code}" \
                            -X POST "$APIGW_URL/rest/apigateway/apis" \
                            -u "$APIGW_AUTH_USR:$APIGW_AUTH_PSW" \
                            -H "Accept: application/json" \
                            -F "apiDefinition=@swagger_spec.json;type=application/json" \
                            -F "apiName=$API_NAME" \
                            -F "apiVersion=$API_VERSION" \
                            -F "type=openapi" \
                            -F "apiType=REST" || true)

                        echo "HTTP Code: $http_code"
                        cat resp.json

                        if [[ "$http_code" -lt 200 || "$http_code" -ge 300 ]]; then
                            echo "API import failed"
                            exit 1
                        fi

                        # ดึง apiId ออกมาเพื่อใช้ในขั้นตอนถัดไป (Activation)
                        API_ID=$(jq -r '.id' resp.json)
                        echo "$API_ID" > api_id.txt
                        echo "Imported API ID: $API_ID"
                    '''
                }
            }
        }

        stage('Activate API') {
            steps {
                echo "Activating API..."
                withCredentials([usernamePassword(
                    credentialsId: 'apigw-admin-password',
                    usernameVariable: 'APIGW_AUTH_USR',
                    passwordVariable: 'APIGW_AUTH_PSW'
                )]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        API_ID=$(cat api_id.txt)
                        
                        # สั่ง Activate API เพื่อให้พร้อมใช้งานทันที (เลียนแบบ set-active: true)
                        curl -sS -X PUT "$APIGW_URL/rest/apigateway/apis/$API_ID/activate" \
                             -u "$APIGW_AUTH_USR:$APIGW_AUTH_PSW" \
                             -H "Accept: application/json"
                        
                        echo "API $API_ID is now Active ✅"
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'rm -f swagger_spec.json resp.json api_id.txt || true'
        }
    }
}