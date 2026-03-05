pipeline {
    agent any

    environment {
        API_SERVER_URL = "http://172.188.16.48:8000/openapi.json"
        APIGW_URL      = "http://20.198.251.142:5555"
        API_NAME       = "customer-erp-API"
        API_VERSION    = "1.0.${BUILD_NUMBER}"
    }

    stages {
        stage('Check Connectivity') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'apigw-admin-password', usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh 'curl -fsSI "$API_SERVER_URL" >/dev/null'
                    sh 'curl -fsSI -u "$U:$P" "$APIGW_URL/rest/apigateway/health" >/dev/null'
                }
            }
        }

        stage('Extract & Patch Spec') {
            steps {
                echo "Downloading and Scrubbing Spec..."
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    curl -fsSL "$API_SERVER_URL" -o raw.json
                    
                    # 1. เปลี่ยนเป็น 3.0.0
                    # 2. ลบภาษาไทยใน info และ tags ทั้งหมด
                    # 3. แก้ไข anyOf [type: null] ที่เป็นปัญหาหลักของ 3.1 -> 3.0
                    jq '.openapi = "3.0.0" | 
                        .info.title = "'${API_NAME}'" |
                        .info.description = "Automated Import" |
                        .info.version = "'${API_VERSION}'" |
                        del(.info.contact) |
                        (.. | select(type == "object" and has("anyOf"))) |= (
                            if .anyOf | any(.type == "null") then 
                                . + {"nullable": true} | del(.anyOf) 
                            else . end
                        ) |
                        del(.paths[][][].tags) |
                        del(.tags)' raw.json > swagger_spec.json

                    echo "Patched Spec (First 15 lines):"
                    head -n 15 swagger_spec.json
                '''
            }
        }

        stage('Push to IBM API Gateway') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'apigw-admin-password', usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        
                        # เพิ่มความชัวร์ด้วยการส่งเป็นไฟล์แนบพร้อมระบุชนิดข้อมูลชัดเจน
                        resp=$(curl -sS -w "\n%{http_code}" \
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

                        echo "HTTP Code: $http_code"
                        echo "Response: $body"

                        if [[ "$http_code" -lt 200 || "$http_code" -ge 300 ]]; then
                            exit 1
                        fi
                        
                        echo $(echo "$body" | jq -r .id) > api_id.txt
                    '''
                }
            }
        }

        stage('Activate API') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'apigw-admin-password', usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        API_ID=$(cat api_id.txt)
                        curl -sS -X PUT "$APIGW_URL/rest/apigateway/apis/$API_ID/activate" \
                             -u "$U:$P" -H "Accept: application/json"
                        echo "API $API_ID Activated ✅"
                    '''
                }
            }
        }
    }

    post {
        always { sh 'rm -f raw.json swagger_spec.json api_id.txt || true' }
    }
}