pipeline {
    agent any

    environment {
        // --- Configuration ---
        API_SERVER_URL = "http://172.188.16.48:8000/openapi.json"
        APIGW_URL      = "http://20.198.251.142:5555"
        API_NAME       = "customer-erp-API"
        API_VERSION    = "1.0.${BUILD_NUMBER}"
        CRED_ID        = "apigw-admin-password"
    }

    stages {
        stage('1. Check Connectivity') {
            steps {
                echo "🔍 Checking API Server and Gateway connectivity..."
                withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        curl -fsSI "$API_SERVER_URL" >/dev/null
                        curl -fsSI -u "$U:$P" "$APIGW_URL/rest/apigateway/health" >/dev/null
                    '''
                }
            }
        }

        stage('2. Extract & Patch Spec') {
            steps {
                echo "📝 Downloading and scrubbing OpenAPI spec (Fixing 3.1.0 -> 3.0.0)..."
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    curl -fsSL "$API_SERVER_URL" -o raw.json
                    
                    # เลียนแบบความฉลาดของ GitHub Actions: 
                    # - บังคับ 3.0.0 
                    # - ล้างภาษาไทยใน description/tags ออกเพื่อป้องกัน 400 Bad Request
                    # - แก้ไข schema nullability ให้ Gateway อ่านออก
                    jq '.openapi = "3.0.0" | 
                        .info.title = "'${API_NAME}'" |
                        .info.version = "'${API_VERSION}'" |
                        .info.description = "Automated build via Jenkins" |
                        del(.info.contact) |
                        del(.tags) |
                        del(.paths[][][].tags) |
                        (.. | select(type == "object" and has("anyOf"))) |= (
                            if .anyOf | any(.type == "null") then 
                                . + {"nullable": true} | del(.anyOf) 
                            else . end
                        )' raw.json > swagger_spec.json

                    echo "✅ Patch completed. Version: ${API_VERSION}"
                '''
            }
        }

        stage('3. Register to API Gateway') {
            steps {
                echo "🚀 Registering/Updating API on IBM Gateway..."
                withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        
                        # ใช้ Multipart Form เหมือนการลากไฟล์วางหน้าเว็บ (Manual)
                        resp=$(curl -sS -w "\\n%{http_code}" \
                            -X POST "$APIGW_URL/rest/apigateway/apis" \
                            -u "$U:$P" \
                            -H "Accept: application/json" \
                            -F "apiDefinition=@swagger_spec.json;type=application/json" \
                            -F "apiName=$API_NAME" \
                            -F "apiVersion=$API_VERSION" \
                            -F "type=openapi" \
                            -F "apiType=REST")

                        http_code=$(echo "$resp" | tail -n1)
                        body=$(echo "$resp" | sed '$d')

                        echo "📡 HTTP Status: $http_code"
                        
                        if [[ "$http_code" -lt 200 || "$http_code" -ge 300 ]]; then
                            echo "❌ Error details: $body"
                            exit 1
                        fi
                        
                        # ดึง API ID ออกมาใช้ในขั้นตอน Activate
                        API_ID=$(echo "$body" | jq -r .id)
                        echo "$API_ID" > api_id.txt
                        echo "✅ Registered Success! API ID: $API_ID"
                    '''
                }
            }
        }

        stage('4. Set Active') {
            steps {
                echo "⚡ Activating API (Making it ready to use)..."
                withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        API_ID=$(cat api_id.txt)
                        
                        # สั่ง Activate ทันที เลียนแบบ set-active: true
                        curl -sS -X PUT "$APIGW_URL/rest/apigateway/apis/$API_ID/activate" \
                             -u "$U:$P" -H "Accept: application/json"
                        
                        echo "✅ API $API_ID is now ACTIVE and ready for requests."
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline Finished Successfully!"
        }
        failure {
            echo "🚨 Pipeline Failed. Please check the logs above."
        }
        always {
            sh 'rm -f raw.json swagger_spec.json api_id.txt || true'
        }
    }
}