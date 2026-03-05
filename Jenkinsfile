pipeline {
    agent any

    environment {
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

        stage('2. Extract & Convert to Swagger 2.0') {
            steps {
                echo "📝 Downloading and converting spec to Swagger 2.0 (Most stable for IBM)..."
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    curl -fsSL "$API_SERVER_URL" -o raw.json
                    
                    # แปลงโครงสร้างจาก 3.x เป็น 2.0 (Swagger)
                    # - เปลี่ยน 'openapi' เป็น 'swagger: 2.0'
                    # - ย้าย 'components/schemas' ไปที่ 'definitions'
                    # - ล้างภาษาไทยและฟิลด์ที่ไม่จำเป็นออกทั้งหมด
                    jq '
                    .swagger = "2.0" | 
                    del(.openapi) |
                    .info.title = "'${API_NAME}'" |
                    .info.version = "'${API_VERSION}'" |
                    .info.description = "Automated Swagger 2.0 Build" |
                    .definitions = .components.schemas |
                    del(.components) |
                    (.. | select(type == "object" and has("$ref"))) |= (
                        if ."$ref" | startswith("#/components/schemas/") then 
                            ."$ref" |= sub("#/components/schemas/"; "#/definitions/") 
                        else . end
                    ) |
                    (if .paths then .paths | map_values(map_values(
                        (if has("summary") then .summary = "Endpoint" else . end) |
                        (if has("description") then .description = "Description" else . end) |
                        del(.tags?)
                    )) else . end) |
                    del(.tags) | del(.info.contact)
                    ' raw.json > swagger_spec.json

                    echo "✅ Conversion to Swagger 2.0 completed."
                    head -n 20 swagger_spec.json
                '''
            }
        }

        stage('3. Register to API Gateway') {
            steps {
                echo "🚀 Registering API on IBM Gateway..."
                withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        
                        resp=$(curl -sS -w "\\n%{http_code}" \
                            -X POST "$APIGW_URL/rest/apigateway/apis" \
                            -u "$U:$P" \
                            -H "Accept: application/json" \
                            -F "apiDefinition=@swagger_spec.json;type=application/json" \
                            -F "apiName=$API_NAME" \
                            -F "apiVersion=$API_VERSION" \
                            -F "type=swagger" \
                            -F "apiType=REST")

                        http_code=$(echo "$resp" | tail -n1)
                        body=$(echo "$resp" | sed '$d')

                        echo "📡 HTTP Status: $http_code"
                        
                        if [[ "$http_code" -lt 200 || "$http_code" -ge 300 ]]; then
                            echo "❌ Error details: $body"
                            exit 1
                        fi
                        
                        API_ID=$(echo "$body" | jq -r .id)
                        echo "$API_ID" > api_id.txt
                        echo "✅ Registered Success! API ID: $API_ID"
                    '''
                }
            }
        }

        stage('4. Set Active') {
            steps {
                echo "⚡ Activating API..."
                withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        API_ID=$(cat api_id.txt)
                        curl -sS -X PUT "$APIGW_URL/rest/apigateway/apis/$API_ID/activate" \
                             -u "$U:$P" -H "Accept: application/json"
                        echo "✅ API $API_ID is now ACTIVE."
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'rm -f raw.json swagger_spec.json api_id.txt || true'
        }
    }
}