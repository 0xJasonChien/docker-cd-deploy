# Deploy Docker To VPS
這是我為自己的 Side Project 所開發的一個 GitHub Action，目的是簡化 Docker 專案部署到 VPS 的流程。
每次推送程式碼到 main 分支時，它會自動透過 SSH 登入遠端伺服器、拉取最新程式碼，然後使用 Docker Compose 建置並啟動容器。
