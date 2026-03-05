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
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        curl -fsSI "$API_SERVER_URL" >/dev/null
                        curl -fsSI -u "$U:$P" "$APIGW_URL/rest/apigateway/health" >/dev/null
                    '''
                }
            }
        }

        stage('Extract & Patch Spec') {
            steps {
                echo "Downloading and Scrubbing Spec for webMethods..."
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    curl -fsSL "$API_SERVER_URL" -o raw.json
                    
                    # 1. บังคับเป็น 3.0.0 (เพราะ 3.1.0 มักจะทำ API พัง)
                    # 2. ล้างอักขระพิเศษภาษาไทยใน info และลบส่วนที่ Gateway มักประมวลผลพลาด (เช่น tags, contact)
                    # 3. แก้ไข anyOf: [type: null] เป็น nullable: true (จุดสำคัญที่ทำให้ 400 Bad Request)
                    jq '.openapi = "3.0.0" | 
                        .info.title = "'${API_NAME}'" |
                        .info.description = "Automated Build" |
                        .info.version = "'${API_VERSION}'" |
                        del(.info.contact) |
                        del(.tags) |
                        del(.paths[][][].tags) |
                        (.. | select(type == "object" and has("anyOf"))) |= (
                            if .anyOf | any(.type == "null") then 
                                . + {"nullable": true} | del(.anyOf) 
                            else . end
                        )' raw.json > swagger_spec.json

                    echo "Preview Patched JSON (Top):"
                    head -n 20 swagger_spec.json
                '''
            }
        }

        stage('Push to IBM API Gateway') {
            steps {
                echo "Registering API..."
                withCredentials([usernamePassword(credentialsId: 'apigw-admin-password', usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        
                        # ส่งไฟล์แบบ Multipart Form (เลียนแบบหน้าเว็บ Manual Import)
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
                        
                        if [[ "$http_code" -lt 200 || "$http_code" -ge 300 ]]; then
                            echo "Error Response: $body"
                            exit 1
                        fi
                        
                        API_ID=$(echo "$body" | jq -r .id)
                        echo "Registered API ID: $API_ID"
                        echo "$API_ID" > api_id.txt
                    '''
                }
            }
        }

        stage('Activate API') {
            steps {
                echo "Activating API (Set Active)..."
                withCredentials([usernamePassword(credentialsId: 'apigw-admin-password', usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        API_ID=$(cat api_id.txt)
                        
                        # สั่ง Activate ทันทีเพื่อให้พร้อมใช้งาน (เลียนแบบ set-active: true ใน GitHub)
                        curl -sS -X PUT "$APIGW_URL/rest/apigateway/apis/$API_ID/activate" \
                             -u "$U:$P" -H "Accept: application/json"
                        
                        echo "API $API_ID is now ACTIVE ✅"
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