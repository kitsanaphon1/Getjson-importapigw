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

        stage('2. Extract & Convert to Pure Swagger 2.0') {
            steps {
                echo "📝 Converting Spec to Pure Swagger 2.0 (Handling requestBody & Responses)..."
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    curl -fsSL "$API_SERVER_URL" -o raw.json
                    
                    # แปลงโครงสร้างแบบละเอียด:
                    # 1. เปลี่ยนเป็น Swagger 2.0 และย้าย Definitions
                    # 2. แก้ปัญหา Response Schema (ดึงออกจาก content)
                    # 3. แก้ปัญหา requestBody (แปลงเป็น parameters + in: body)
                    # 4. ล้างภาษาไทยและ Tags ออกทั้งหมด
                    jq '
                    .swagger = "2.0" | 
                    del(.openapi) |
                    .info.title = "'${API_NAME}'" |
                    .info.version = "'${API_VERSION}'" |
                    .info.description = "Automated Swagger 2.0 Build" |
                    .definitions = .components.schemas |
                    del(.components) |
                    
                    # แก้ไข Reference ให้ชี้ไป definitions
                    (.. | select(type == "object" and has("$ref"))) |= (
                        if ."$ref" | startswith("#/components/schemas/") then 
                            ."$ref" |= sub("#/components/schemas/"; "#/definitions/") 
                        else . end
                    ) |
                    
                    # แก้ไข Paths: จัดการทั้ง Responses และ requestBody
                    (if .paths then .paths | map_values(map_values(
                        (if has("summary") then .summary = "Endpoint" else . end) |
                        (if has("description") then .description = "Description" else . end) |
                        del(.tags?) |
                        
                        # จัดการ Responses (ดึง schema ออกจาก content)
                        (.responses | map_values(
                            if .content."application/json".schema then 
                                .schema = .content."application/json".schema | del(.content)
                            else . end
                        )) as $resp | .responses = $resp |
                        
                        # จัดการ requestBody (เปลี่ยนเป็น parameters แบบ body ตามมาตรฐาน 2.0)
                        if .requestBody then
                            .parameters = [{
                                "in": "body",
                                "name": "body",
                                "required": true,
                                "schema": .requestBody.content."application/json".schema
                            }] | del(.requestBody)
                        else . end
                    )) else . end) |
                    
                    # ล้าง anyOf [null] และ title/examples
                    (.. | select(type == "object" and has("anyOf"))) |= (
                        if .anyOf | any(.type == "null") then 
                            .type = "string" | .x_nullable = true | del(.anyOf)
                        else . end
                    ) |
                    (.. | select(type == "object")) |= del(.title?, .examples?) |
                    del(.tags) | del(.info.contact)
                    ' raw.json > swagger_spec.json

                    echo "✅ Deep Conversion to Swagger 2.0 completed."
                    head -n 30 swagger_spec.json
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