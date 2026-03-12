name: cicd

on:
  push:
    branches:
      - main
    tags:
      - 'v*'

jobs:
  update-dev:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: development
    steps:
      - uses: actions/checkout@v4

      # ขั้นตอนที่ 1: ไปดึง Spec จาก FastAPI Server ของคุณ
      - name: Fetch OpenAPI Spec from FastAPI
        run: |
          curl -s https://petstore.swagger.io/v2/swagger.json -o swagger.json
          # ตรวจสอบว่าได้ไฟล์มาจริงไหม
          ls -l swagger.json

      # ขั้นตอนที่ 2: Import เข้า IBM API Gateway บน Azure
      - name: Register API to IBM APIGW
        uses: jiridj/wm-apigw-actions-register-api@v1
        with: 
          apigw-url: http://20.198.251.142:5555
          apigw-username: ${{ secrets.APIGW_USERNAME }}
          apigw-password: ${{ secrets.APIGW_PASSWORD }}
          api-spec: swagger.json
          set-active: true
          debug: true







jobs:
  update-dev:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        api_url: [
          "https://petstore.swagger.io/v2/swagger.json",
          "https://raw.githubusercontent.com/kubernetes/kubernetes/master/api/openapi-spec/swagger.json",
          "https://api.apis.guru/v2/specs/instagram.com/1.0.0/swagger.json",
          "https://api.apis.guru/v2/specs/azure.com/storage/2019-06-01/swagger.json",
          "https://api.apis.guru/v2/specs/1forge.com/0.0.1/swagger.json",
          # ... ใส่ให้ครบ 17 ตัว คั่นด้วยลูกน้ำ (,)
        ]
    steps:
      - uses: actions/checkout@v4
      - name: Fetch Spec
        run: curl -s ${{ matrix.api_url }} -o swagger.json
      - name: Register API to IBM APIGW
        uses: jiridj/wm-apigw-actions-register-api@v1
        with: 
          apigw-url: http://20.198.251.142:5555
          apigw-username: ${{ secrets.APIGW_USERNAME }}
          apigw-password: ${{ secrets.APIGW_PASSWORD }}
          api-spec: swagger.json
          set-active: true





api import 14 list 