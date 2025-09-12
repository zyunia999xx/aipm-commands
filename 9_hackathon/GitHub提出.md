# 13 GitHub提出（AIPMハッカソン）

## 目的
Stock側の提出フォルダをリポジトリ化してPushするとともに、共有用リポジトリ（Share）にも自分用フォルダを作ってコピーし、submissionブランチを作成してPull Request（PR）を作成します。

## 事前条件
- GitHub CLI(gh) が利用可能でログイン済みであること
- 初期化で `project_init.yaml` を作成済（share_repo, share_local_dir を参照）

## 実行手順
```yaml
- trigger: "(提出_GitHub提出|SubmitGitHub)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns:['Stock/**/submissions/**/README_submission.md','**/project_init.yaml']))}}"]
      instructions: |
        Program/Project/submit_folder/share_repo/participant_name/github_id の初期値候補を抽出し、displayで提示してください。
      store_as: "auto_github_submit_seed"
    - name: "prefill_github_submit_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_github_submit_seed}}
    - name: "collect_submit_inputs"
      action: "ask_questions_with_template"
      template: |
        === GitHub提出メタ ===
        1. Program（既定: AIPM_Hackathon）
        →
        2. Project（アプリ名）
        →
        3. 提出フォルダ名（例: submit_{{today}}）
        →
        4. 共有Repo（既定: libyn-inc/aipm-hackathon-share）
        →
        5. 参加者名（自分のフォルダ名に使用）
        →
        6. GitHub ID（個別提出Repoの所有者）
        →
        =====================================
    
    - name: "wait_submit"
      action: "wait_for_all_answers"
    
    - name: "confirm_git_ops"
      action: "confirm"
      message: "Stockの提出フォルダをGit初期化→コミット→Pushし、Shareリポジトリへは submission ブランチを作成して PR を作ります。よろしいですか？"
    
    - name: "show_parallel_shells"
      action: "display"
      content: |
        💻 実行環境メモ（Windows対応・手動時の参考）

        Bash (macOS/Linux):
        ```bash
        cd "Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}}"
        git init && git add . && git commit -m "feat: initial submission" || true
        gh repo create "{{project_id}}-{{submit_folder}}" --private --source=. --push --disable-wiki --confirm || true
        git push -u origin HEAD || true
        ```

        PowerShell (Windows/macOSのpwsh):
        ```powershell
        Set-Location "Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}}"
        git init; git add .; git commit -m "feat: initial submission" 2>$null
        gh repo create "{{project_id}}-{{submit_folder}}" --private --source=. --push --disable-wiki --confirm 2>$null
        git push -u origin HEAD 2>$null
        ```

    # 1) Stock側をGitリポジトリ化（ローカル提出用）
    - name: "init_local_repo"
      action: "execute_shell"
      command: |
        cd "Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}}" && \
        git init && \
        git add . && \
        git commit -m "feat: initial submission" || true
    
    # 2) GitHub新規作成 or 既存リモート設定（個別提出リポジトリ）
    - name: "create_or_set_remote"
      action: "execute_shell"
      command: |
        cd "Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}}" && \
        gh repo create "{{project_id}}-{{submit_folder}}" --private --source=. --push --disable-wiki --confirm || \
        (git remote add origin "https://github.com/{{github_id}}/{{project_id}}-{{submit_folder}}.git" 2>/dev/null || true) && \
        git push -u origin HEAD || true
    
    # 3) Shareリポジトリをローカルに準備（なければClone）
    - name: "prepare_share_repo"
      action: "execute_shell"
      command: |
        mkdir -p "Share" && \
        if [ ! -d "{{share_local_dir}}/.git" ]; then gh repo clone "{{share_repo}}" "{{share_local_dir}}"; fi && \
        cd "{{share_local_dir}}" && \
        git fetch --all --prune && \
        git pull --rebase || true
    
    # 4) 参加者フォルダを作成し、提出物をコピー（まだコミットしない）
    - name: "copy_to_share"
      action: "execute_shell"
      command: |
        mkdir -p "{{share_local_dir}}/{{participant_name}}" && \
        rm -rf "{{share_local_dir}}/{{participant_name}}/{{project_id}}/{{submit_folder}}" 2>/dev/null || true && \
        mkdir -p "{{share_local_dir}}/{{participant_name}}/{{project_id}}" && \
        cp -R "Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}}" "{{share_local_dir}}/{{participant_name}}/{{project_id}}/"
    
    # 5) submissionブランチを作成し、コミット→Push
    - name: "create_branch_and_push"
      action: "execute_shell"
      command: |
        BR="submission/{{participant_name}}/{{project_id}}/{{submit_folder}}" && \
        cd "{{share_local_dir}}" && \
        git checkout -b "$BR" || git checkout -B "$BR" && \
        git add . && \
        git commit -m "chore: add submission {{participant_name}}/{{project_id}}/{{submit_folder}}" || true && \
        git push -u origin "$BR"
    
    # 6) PR作成（base: main が無ければ master を試行）
    - name: "create_pull_request"
      action: "execute_shell"
      command: |
        BR="submission/{{participant_name}}/{{project_id}}/{{submit_folder}}" && \
        cd "{{share_local_dir}}" && \
        gh pr create --title "submission: {{participant_name}}/{{project_id}}/{{submit_folder}}" \
                      --body "Auto-submission via AIPM Commands.\nProject: {{project_id}}\nSubmit: {{submit_folder}}\nPath: {{participant_name}}/{{project_id}}/{{submit_folder}}" \
                      --base main --head "$BR" || \
        gh pr create --title "submission: {{participant_name}}/{{project_id}}/{{submit_folder}}" \
                      --body "Auto-submission via AIPM Commands.\nProject: {{project_id}}\nSubmit: {{submit_folder}}\nPath: {{participant_name}}/{{project_id}}/{{submit_folder}}" \
                      --base master --head "$BR" || true
    
    - name: "show_pr_url"
      action: "execute_shell"
      command: |
        cd "{{share_local_dir}}" && gh pr view --json url -q .url | cat
    
    - name: "notify"
      action: "display"
      content: |
        ✅ GitHub提出（PR作成）まで完了しました。
        - 個別提出Repo: {{project_id}}-{{submit_folder}}
        - Share: {{share_repo}} の `{{participant_name}}/{{project_id}}/{{submit_folder}}` 配下
        - PR: 上記で表示されたURLをご確認ください（レビュー/マージは運営側で実施）
```

## 備考
- Shareリポが無い場合は、運営へ権限設定を依頼してください。

