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
        stage('1. Prepare Spec') {
            steps {
                echo "📥 Downloading and Cleaning Spec for webMethods..."
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    curl -fsSL "$API_SERVER_URL" -o raw.json
                    
                    # 1. บังคับเป็น OpenAPI 3.0.0 (เพราะ 3.1.0 มักจะทำ API พังผ่าน REST)
                    # 2. แก้ปัญหา anyOf: [type: null] เป็น nullable: true (จุดที่ Manual ผ่านแต่ curl มักไม่ผ่าน)
                    # 3. ลบภาษาไทยใน info และ tags ออกทั้งหมดเพื่อป้องกันปัญหา Encoding
                    jq '.openapi = "3.0.0" | 
                        .info.title = "'${API_NAME}'" |
                        .info.version = "'${API_VERSION}'" |
                        .info.description = "Automated Import" |
                        del(.info.contact) |
                        del(.tags) |
                        (if .paths then .paths | map_values(map_values(del(.tags?))) else . end) |
                        (.. | select(type == "object" and has("anyOf"))) |= (
                            if .anyOf | any(.type == "null") then 
                                . + {"nullable": true} | del(.anyOf) 
                            else . end
                        )' raw.json > swagger_spec.json
                '''
            }
        }

        stage('2. Register to Gateway') {
            steps {
                echo "🚀 Uploading Spec..."
                withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        
                        # เพิ่ม -H "Expect:" และระบุ charset=utf-8 เพื่อเลียนแบบ Browser
                        resp=$(curl -sS -k -w "\\n%{http_code}" \
                            -X POST "$APIGW_URL/rest/apigateway/apis" \
                            -u "$U:$P" \
                            -H "Accept: application/json" \
                            -H "Expect:" \
                            -F "apiDefinition=@swagger_spec.json;type=application/json;charset=utf-8" \
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
                        
                        API_ID=$(echo "$body" | jq -r .id)
                        echo "$API_ID" > api_id.txt
                        echo "✅ Success! API ID: $API_ID"
                    '''
                }
            }
        }

        stage('3. Set Active') {
            steps {
                echo "⚡ Activating API..."
                withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        API_ID=$(cat api_id.txt)
                        curl -sS -k -X PUT "$APIGW_URL/rest/apigateway/apis/$API_ID/activate" \
                             -u "$U:$P" -H "Accept: application/json"
                        echo "✅ API is now ACTIVE"
                    '''
                }
            }
        }
    }

    post {
        always { sh 'rm -f raw.json swagger_spec.json api_id.txt || true' }
    }
}