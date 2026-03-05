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
                echo "📥 Downloading Spec..."
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    curl -fsSL "$API_SERVER_URL" -o raw.json
                    
                    # แก้แค่ Version ให้ตรงกับ Jenkins Build (เพื่อให้ Manual สวยงาม)
                    jq '.info.version = "'${API_VERSION}'"' raw.json > swagger_spec.json
                '''
            }
        }

        stage('2. Register to Gateway') {
            steps {
                echo "🚀 Uploading like Manual Import..."
                withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        
                        # ใช้ -F แบบระบุ Type ชัดเจนที่สุด เลียนแบบ Browser
                        resp=$(curl -sS -k -w "\\n%{http_code}" \
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
                        
                        API_ID=$(echo "$body" | jq -r .id)
                        echo "$API_ID" > api_id.txt
                        echo "✅ Success! API ID: $API_ID"
                    '''
                }
            }
        }

        stage('3. Set Active') {
            steps {
                withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        API_ID=$(cat api_id.txt)
                        curl -sS -k -X PUT "$APIGW_URL/rest/apigateway/apis/$API_ID/activate" \
                             -u "$U:$P" -H "Accept: application/json"
                        echo "✅ API is now Active"
                    '''
                }
            }
        }
    }

    post {
        always { sh 'rm -f raw.json swagger_spec.json api_id.txt || true' }
    }
}