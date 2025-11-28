name: Full Upstream Sync (Best Version with Conflict Alerts)

on:
  schedule:
    - cron: "0 18 * * *"      # 每天 18:00
  workflow_dispatch:           # 可手动触发

env:
  UPSTREAM_REPO: "https://github.com/frankiejun/wxpush.git"
  SYNC_BRANCH_PREFIX: "sync-upstream"
  SYNC_BRANCHES: "main dev release"   # 可自由调整
  TELEGRAM_ENABLE: "true"
  DISCORD_ENABLE: "false"

jobs:
  sync:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure git
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

      - name: Add upstream remote
        run: |
          if git remote | grep -q '^upstream$'; then
            echo "Upstream exists."
          else
            git remote add upstream $UPSTREAM_REPO
          fi
          git fetch upstream --tags

      - name: Detect default branch
        id: branch
        run: |
          DEFAULT=$(git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@')
          echo "default_branch=$DEFAULT" >> $GITHUB_OUTPUT

      - name: Sync branches
        run: |
          declare -A BRANCH_STATUS
          CONFLICT_BRANCHES=""
          BRANCHES=""
          for BR in $SYNC_BRANCHES; do
            echo ">>> Syncing branch: $BR"
            if ! git branch -a | grep -q "origin/$BR"; then
              echo "Branch $BR not found, skip."
              BRANCH_STATUS[$BR]="⚠️ Not Found"
              continue
            fi
            SYNC_BRANCH="${SYNC_BRANCH_PREFIX}-${BR}-$(date +%Y%m%d-%H%M%S)"
            git checkout -b "$SYNC_BRANCH" "origin/$BR"
            if git merge "upstream/$BR" --no-edit; then
              BRANCH_STATUS[$BR]="✅ Synced"
            else
              BRANCH_STATUS[$BR]="❌ Conflict"
              CONFLICT_BRANCHES="$CONFLICT_BRANCHES $BR"
            fi
            git push origin "$SYNC_BRANCH"
            BRANCHES="$BRANCHES $SYNC_BRANCH"
          done
          echo "sync_branches=$BRANCHES" >> $GITHUB_ENV
          echo "conflict_branches=$CONFLICT_BRANCHES" >> $GITHUB_ENV
          # 保存分支状态
          STATUS_OUTPUT=""
          for B in "${!BRANCH_STATUS[@]}"; do
            STATUS_OUTPUT+="$B : ${BRANCH_STATUS[$B]}\n"
          done
          echo -e "$STATUS_OUTPUT" > sync_status.txt
          echo "SYNC_STATUS_FILE=sync_status.txt" >> $GITHUB_ENV

      - name: Show Branch Sync Status
        run: |
          echo "===== Branch Sync Status ====="
          cat ${{ env.SYNC_STATUS_FILE }}
          echo "=============================="

      - name: Create Pull Requests
        uses: peter-evans/create-pull-request@v6
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          title: "🔄 Sync from Upstream"
          body: |
            自动从 upstream 同步。
            同步分支状态：
            ```
            $(cat ${{ env.SYNC_STATUS_FILE }})
            ```
            若出现冲突，请按 Issue 提示处理。
          branch: ${{ env.sync_branches }}
          labels: ["sync", "automated"]

      - name: Detect merge conflicts
        run: |
          if [ -n "${{ env.conflict_branches }}" ]; then
            echo "conflict=true" >> $GITHUB_ENV
          else
            echo "conflict=false" >> $GITHUB_ENV
          fi

      - name: Create Conflict Issue
        if: ${{ env.conflict == 'true' }}
        uses: peter-evans/create-issue-from-file@v5
        with:
          title: "❗ Upstream Sync Merge Conflict"
          content-filepath: .github/CONFLICT_TEMPLATE.md
          labels: conflict

      - name: Sync tags
        run: git push origin --tags

      - name: Auto merge PR (if no conflict)
        if: ${{ env.conflict == 'false' }}
        uses: peter-evans/enable-pull-request-automerge@v3
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          merge-method: squash

      - name: Notify Telegram
        if: ${{ env.TELEGRAM_ENABLE == 'true' }}
        run: |
          if [ "${{ env.conflict }}" == "true" ]; then
            MSG="⚠️ Upstream Sync 完成，但以下分支有冲突: ${{ env.conflict_branches }}\n\n同步状态：\n$(cat ${{ env.SYNC_STATUS_FILE }})"
          else
            MSG="✅ Upstream Sync 完成，所有分支同步成功。\n\n同步状态：\n$(cat ${{ env.SYNC_STATUS_FILE }})"
          fi
          curl -s -X POST "https://api.telegram.org/bot${{ secrets.TELEGRAM_BOT_TOKEN }}/sendMessage" \
          -d chat_id="${{ secrets.TELEGRAM_CHAT_ID }}" \
          -d text="$MSG"
