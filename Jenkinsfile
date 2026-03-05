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

curl_call() {
  curl -sS -k -w "\n%{http_code}" "$@"
}

case "$cmd" in
  import)
    spec="${1:-swagger_spec.json}"
    [[ -f "$spec" ]] || { echo "Spec not found: $spec" >&2; exit 2; }

    need_env API_NAME
    need_env API_VERSION

    resp="$(curl_call \
      -X POST "$APIGW_URL/rest/apigateway/apis" \
      -u "$U:$P" \
      -H "Accept: application/json" \
      -F "apiDefinition=@${spec};type=application/json" \
      -F "apiName=$API_NAME" \
      -F "apiVersion=$API_VERSION" \
      -F "type=openapi" \
      -F "apiType=REST")"

    http_code="$(echo "$resp" | tail -n1)"
    body="$(echo "$resp" | sed '$d')"

    if [[ "$http_code" -lt 200 || "$http_code" -ge 300 ]]; then
      echo "❌ Import failed (HTTP $http_code)" >&2
      echo "----- Response body -----" >&2
      echo "$body" >&2
      echo "-------------------------" >&2
      exit 1
    fi

    echo "$body"
    ;;

  activate)
    api_id="${1:-}"
    [[ -n "$api_id" ]] || { echo "Usage: $0 activate <api_id>" >&2; exit 2; }

    resp="$(curl_call \
      -X PUT "$APIGW_URL/rest/apigateway/apis/$api_id/activate" \
      -u "$U:$P" \
      -H "Accept: application/json")"

    http_code="$(echo "$resp" | tail -n1)"
    body="$(echo "$resp" | sed '$d')"

    if [[ "$http_code" -lt 200 || "$http_code" -ge 300 ]]; then
      echo "❌ Activate failed (HTTP $http_code)" >&2
      echo "----- Response body -----" >&2
      echo "$body" >&2
      echo "-------------------------" >&2
      exit 1
    fi

    echo "$body"
    ;;

  *)
    echo "Usage: $0 import [spec_file] | $0 activate <api_id>" >&2
    exit 2
    ;;
esac
EOF

          chmod +x apigw-cli.sh
        '''
      }
    }

    // ✅ แก้ Stage นี้เท่านั้น
    stage('1. Prepare Spec') {
      steps {
        echo "📥 Deep Cleaning Spec (OAS3 valid)..."
        sh '''#!/usr/bin/env bash
          set -euo pipefail
          curl -fsSL "$API_SERVER_URL" -o raw.json

          jq '
            .openapi = "3.0.0" |
            .info.title = "'${API_NAME}'" |
            .info.version = "'${API_VERSION}'" |
            del(.tags) |
            del(.info.contact) |

            # ทำให้ operation/response ถูกต้องตาม OAS3
            (if .paths then
              .paths |= map_values(
                map_values(
                  (if has("summary") then .summary="Endpoint" else . end) |
                  (if has("description") then .description="Desc" else . end) |
                  del(.tags?) |

                  # response ต้องมี description (บาง gateway ซีเรียส)
                  (if has("responses") then
                    .responses |= map_values(
                      (if has("description") then . else . + {"description":"OK"} end)
                    )
                   else . end)
                )
              )
             else . end) |

            # anyOf null -> nullable (OAS3.0)
            (.. | select(type=="object" and has("anyOf"))) |= (
              if (.anyOf | any(.type=="null")) then
                . + {"nullable": true} | del(.anyOf)
              else .
              end
            )
          ' raw.json > swagger_spec.json

          test -s swagger_spec.json
        '''
      }
    }

    stage('2. Register to Gateway (CLI)') {
      steps {
        echo "🚀 Registering as ${API_NAME}..."
        withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'U', passwordVariable: 'P')]) {
          sh '''#!/usr/bin/env bash
            set -euo pipefail

            body="$(APIGW_URL="$APIGW_URL" U="$U" P="$P" API_NAME="$API_NAME" API_VERSION="$API_VERSION" \
                   ./apigw-cli.sh import swagger_spec.json)"

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

            APIGW_URL="$APIGW_URL" U="$U" P="$P" ./apigw-cli.sh activate "$API_ID" >/dev/null
            echo "✅ Activated Successfully"
          '''
        }
      }
    }
  }

  post {
    always { sh 'rm -f raw.json swagger_spec.json api_id.txt apigw-cli.sh || true' }
  }
}