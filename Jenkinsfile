pipeline {
    agent any

    environment {
        API_SERVER_URL = "http://172.188.16.48:8000/openapi.json"
        APIGW_URL      = "http://20.198.251.142:5555"
        API_NAME       = "customer-erp-new-${BUILD_NUMBER}"
        API_VERSION    = "1.0.${BUILD_NUMBER}"
        CRED_ID        = "apigw-admin-password"
    }

    stages {

        stage('0. Prepare CLI') {
            steps {
                echo "🧰 Preparing APIGW CLI wrapper..."
                sh '''#!/usr/bin/env bash
                    set -euo pipefail

                    cat > apigw-cli.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

cmd="${1:-}"; shift || true

need_env() {
  local k="$1"
  if [[ -z "${!k:-}" ]]; then
    echo "Missing env: $k" >&2
    exit 2
  fi
}

need_env APIGW_URL
need_env U
need_env P

case "$cmd" in
  import)
    spec="${1:-swagger_spec.json}"
    [[ -f "$spec" ]] || { echo "Spec not found: $spec" >&2; exit 2; }

    need_env API_NAME
    need_env API_VERSION

    # return: JSON body to stdout
    curl -sS -k \
      -X POST "$APIGW_URL/rest/apigateway/apis" \
      -u "$U:$P" \
      -H "Accept: application/json" \
      -F "apiDefinition=@${spec};type=application/json" \
      -F "apiName=$API_NAME" \
      -F "apiVersion=$API_VERSION" \
      -F "type=openapi" \
      -F "apiType=REST"
    ;;

  activate)
    api_id="${1:-}"
    [[ -n "$api_id" ]] || { echo "Usage: $0 activate <api_id>" >&2; exit 2; }

    curl -sS -k \
      -X PUT "$APIGW_URL/rest/apigateway/apis/$api_id/activate" \
      -u "$U:$P" \
      -H "Accept: application/json"
    ;;

  delete)
    api_id="${1:-}"
    [[ -n "$api_id" ]] || { echo "Usage: $0 delete <api_id>" >&2; exit 2; }

    curl -sS -k \
      -X DELETE "$APIGW_URL/rest/apigateway/apis/$api_id" \
      -u "$U:$P" \
      -H "Accept: application/json"
    ;;

  *)
    cat >&2 <<USAGE
Usage:
  $0 import [spec_file]
  $0 activate <api_id>
  $0 delete <api_id>

Required env:
  APIGW_URL, U, P
Optional env (for import):
  API_NAME, API_VERSION
USAGE
    exit 2
    ;;
esac
EOF

                    chmod +x apigw-cli.sh
                '''
            }
        }

        stage('1. Prepare Spec') {
            steps {
                echo "📥 Deep Cleaning Spec..."
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    curl -fsSL "$API_SERVER_URL" -o raw.json

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

        stage('2. Register to Gateway (CLI)') {
            steps {
                echo "🚀 Registering as ${API_NAME}..."
                withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail

                        # เรียก CLI import (คืนค่าเป็น JSON body)
                        body="$(APIGW_URL="$APIGW_URL" U="$U" P="$P" API_NAME="$API_NAME" API_VERSION="$API_VERSION" \
                               ./apigw-cli.sh import swagger_spec.json)"

                        # เช็คว่าได้ id จริงไหม (กันกรณี error page ไม่ใช่ JSON)
                        api_id="$(echo "$body" | jq -er '.id')"

                        echo "✅ Imported: id=$api_id"
                        echo "$api_id" > api_id.txt
                    '''
                }
            }
        }

        stage('3. Activate (CLI)') {
            steps {
                withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        API_ID="$(cat api_id.txt)"

                        ./apigw-cli.sh activate "$API_ID" \
                          <<< "" >/dev/null 2>&1 || true

                        # เรียกแบบใส่ env ให้ชัวร์ (บาง agent ไม่ส่ง env เข้า script)
                        APIGW_URL="$APIGW_URL" U="$U" P="$P" ./apigw-cli.sh activate "$API_ID" >/dev/null

                        echo "✅ Activated Successfully"
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'rm -f raw.json swagger_spec.json api_id.txt apigw-cli.sh || true'
        }
    }
}