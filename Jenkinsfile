pipeline {
    agent any

    environment {
        API_SERVER_URL = "http://172.188.16.48:8000/openapi.json"
        APIGW_URL      = "http://20.198.251.142:5555"
        // ลองเปลี่ยนชื่อ API เพื่อเลี่ยงการซ้ำในระบบ
        API_NAME       = "customer-erp-new-${BUILD_NUMBER}" 
        API_VERSION    = "1.0.${BUILD_NUMBER}"
        CRED_ID        = "apigw-admin-password"
    }

    stages {
        stage('1. Prepare Spec') {
            steps {
                echo "📥 Deep Cleaning Spec..."
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    curl -fsSL "$API_SERVER_URL" -o raw.json
                    
                    # ล้างทุกอย่างให้เป็นมาตรฐานที่ webMethods ยอมรับแน่นอน
                    # - บังคับ 3.0.0
                    # - ล้างภาษาไทย และ Tags ทั้งหมด
                    # - ดึง schema ออกมาจาก content (จุดตายของ 400)
                    jq '.openapi = "3.0.0" | 
                        .info.title = "'${API_NAME}'" |
                        .info.version = "'${API_VERSION}'" |
                        del(.tags) | del(.info.contact) |
                        (if .paths then .paths | map_values(map_values(
                            (if has("summary") then .summary = "Endpoint" else . end) |
                            (if has("description") then .description = "Desc" else . end) |
                            del(.tags?) |
                            (.responses | map_values(
                                if .content."application/json".schema then 
                                    .schema = .content."application/json".schema | del(.content)
                                else . end
                            )) as $resp | .responses = $resp
                        )) else . end) |
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
                echo "🚀 Registering as ${API_NAME}..."
                withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        
                        # เพิ่ม -v เพื่อดู Header การส่ง (Debug)
                        resp=$(curl -sS -k -v -w "\\n%{http_code}" \
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

                        echo "📡 Status: $http_code"
                        if [[ "$http_code" -lt 200 || "$http_code" -ge 300 ]]; then
                            echo "❌ Error: $body"
                            exit 1
                        fi
                        
                        echo $(echo "$body" | jq -r .id) > api_id.txt
                    '''
                }
            }
        }

        stage('3. Activate') {
            steps {
                withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        API_ID=$(cat api_id.txt)
                        curl -sS -k -X PUT "$APIGW_URL/rest/apigateway/apis/$API_ID/activate" \
                             -u "$U:$P" -H "Accept: application/json"
                        echo "✅ Activated Successfully"
                    '''
                }
            }
        }
    }

    post {
        always { sh 'rm -f raw.json swagger_spec.json api_id.txt || true' }
    }
}