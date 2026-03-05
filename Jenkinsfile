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
                echo "Checking connection to API Server and Gateway..."
                sh '''
                    set -euo pipefail
                    curl -fsSI "$API_SERVER_URL" >/dev/null
                '''

                withCredentials([usernamePassword(
                    credentialsId: 'apigw-admin-password',
                    usernameVariable: 'APIGW_AUTH_USR',
                    passwordVariable: 'APIGW_AUTH_PSW'
                )]) {
                    sh '''
                        set -euo pipefail
                        curl -fsSI -u "$APIGW_AUTH_USR:$APIGW_AUTH_PSW" "$APIGW_URL/rest/apigateway/health" >/dev/null
                    '''
                }
            }
        }

        stage('Extract OpenAPI Spec') {
            steps {
                echo "Downloading OpenAPI Spec..."
                sh '''
                    set -euo pipefail

                    curl -fsSL "$API_SERVER_URL" -o swagger_spec.json

                    echo "File size:"
                    ls -lh swagger_spec.json

                    echo "First 30 lines (debug):"
                    head -n 30 swagger_spec.json

                    echo "Validate JSON via jq:"
                    jq -e . swagger_spec.json >/dev/null

                    echo "Check basic OpenAPI/Swagger keys:"
                    jq -e '((.openapi != null) or (.swagger != null)) and (.info != null) and (.paths != null)' swagger_spec.json >/dev/null
                '''
            }
        }

        stage('Push to IBM API Gateway') {
            steps {
                echo "Importing API via Multipart Form Data..."
                withCredentials([usernamePassword(
                    credentialsId: 'apigw-admin-password',
                    usernameVariable: 'APIGW_AUTH_USR',
                    passwordVariable: 'APIGW_AUTH_PSW'
                )]) {
                    sh '''
                        set -euo pipefail

                        # เรียก import และเก็บทั้ง http code + response body
                        http_code=$(curl -sS -o resp.json -w "%{http_code}" \
                            -X POST "$APIGW_URL/rest/apigateway/apis" \
                            -u "$APIGW_AUTH_USR:$APIGW_AUTH_PSW" \
                            -H "Accept: application/json" \
                            -F "apiDefinition=@swagger_spec.json;type=application/json" \
                            -F "apiName=$API_NAME" \
                            -F "apiVersion=$API_VERSION" \
                            -F "type=openapi" \
                            -F "apiType=REST" || true)

                        echo "HTTP: $http_code"
                        echo "Response:"
                        cat resp.json || true

                        # fail ถ้าไม่ใช่ 2xx
                        if [ "$http_code" -lt 200 ] || [ "$http_code" -ge 300 ]; then
                            echo "API import failed (HTTP $http_code)"
                            exit 1
                        fi

                        # กันกรณี 200 แต่ใน body มี errorDetails
                        if jq -e '.errorDetails? // empty | length > 0' resp.json >/dev/null; then
                            echo "API import failed (errorDetails present)"
                            exit 1
                        fi

                        echo "API import success ✅"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Congratulations! API has been imported successfully.'
        }
        failure {
            echo 'Failed! Please check API Gateway logs for detail.'
        }
        always {
            sh '''
                rm -f swagger_spec.json resp.json || true
            '''
        }
    }
}