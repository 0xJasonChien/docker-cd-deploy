# Deploy Docker To VPS
這是我為自己的 Side Project 所開發的一個 GitHub Action，目的是簡化 Docker 專案部署到 VPS 的流程。
每次推送程式碼到 main 分支時，它會自動透過 SSH 登入遠端伺服器、拉取最新程式碼，然後使用 Docker Compose 建置並啟動容器。

## 開發動機
平常我喜歡用 Docker 部署專案另外一台 VPS 一年的費用很便宜😂，但每次更新都要手動 SSH 進 VPS 實在很麻煩。
這個 Action 幫我自動化了整個流程，讓我可以專心開發，部署交給 GitHub Actions 處理。

## 程式瑪
``` action.yaml
name: "Deploy Docker To VPS"
description: "Deploy Docker image to VPS using SSH"
inputs:
  SSH_HOST:
    description: "SSH host"
    required: true
  SSH_KEY:
    description: "SSH key"
    required: true
  SSH_USER:
    description: "SSH user"
    required: true
  GITHUB_TOKEN:
    description: "GitHub token for authentication"
    required: true
  ENV_FILE:
    description: ".env file contents"
    required: false
    default: ""

runs:
  using: "composite"
  steps:
  - name: Deploy via SSH
    uses: appleboy/ssh-action@v1.0.0
    with:
      host: ${{ inputs.SSH_HOST }}
      username: ${{ inputs.SSH_USER }}
      key: ${{ inputs.SSH_KEY }}
      debug: true # Enable debug logging
      script: |
        # Install Docker if not already installed
        if ! command -v docker &> /dev/null; then
          sudo apt-get update
          sudo apt-get install -y docker.io
          sudo usermod -aG docker ${{ inputs.SSH_USER }}
        fi

        # Start Docker service if not already running
        sudo systemctl start docker
        sudo systemctl enable docker

        # Install Docker Compose if not already installed
        if ! command -v docker-compose &> /dev/null; then
          sudo apt-get install -y docker-compose
        fi

        # Set up project directory
        REPO_NAME=$(basename "${{ github.repository }}")
        PROJECT_DIR="/app/$REPO_NAME/"

        if [ ! -d "$PROJECT_DIR" ]; then
          mkdir -p "$PROJECT_DIR"
          cd "$PROJECT_DIR"
          # Use authentication with git clone
          git clone https://${{ inputs.GITHUB_TOKEN }}@github.com/${{ github.repository }}.git . || { echo "Clone failed"; exit 1; rm -fr "$PROJECT_DIR"; }
        else
          cd "$PROJECT_DIR"
          # Configure git to use token
          git config --global url."https://${{ inputs.GITHUB_TOKEN }}@github.com/".insteadOf "https://github.com/"
          git checkout main
          git fetch
          git reset --hard origin/main || { echo "Pull failed, continuing with existing files"; }
        fi

        # Create .env file if provided
        if [ -n "${{ inputs.ENV_FILE }}" ]; then
          echo "${{ inputs.ENV_FILE }}" > "$PROJECT_DIR/deploy/.env"
        fi

        # Run Docker Compose
        docker compose --env-file "$PROJECT_DIR/deploy/.env" -f "$PROJECT_DIR/deploy/docker-compose.yaml" down --rmi all
        docker compose --env-file "$PROJECT_DIR/deploy/.env" -f "$PROJECT_DIR/deploy/docker-compose.yaml" build --no-cache
        docker compose --env-file "$PROJECT_DIR/deploy/.env" -f "$PROJECT_DIR/deploy/docker-compose.yaml" up -d --force-recreate

        # Optional: Clean up old images and networks
        docker image prune -f
        docker network prune -f

```

## 預期的專案結構
```
.
├── deploy/
│   ├── .env               
│   └── docker-compose.yaml
```
