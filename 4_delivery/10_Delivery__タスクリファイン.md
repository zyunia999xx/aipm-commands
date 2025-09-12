# 10 進捗レビュー: コンテキスト再認識とタスクリファイン（AIPMハッカソン）

## 目的
- 現在の成果物（dev_tasks.yaml / order / runbook / story_map / screen_map / spec / ui 等）を読み直し、進捗を可視化
- 残タスクの優先順位を再構成（依存関係と学習効果を両立）
- 必要に応じてタスク内容を微修正し、確定版を再出力
- 進捗申告を dev_tasks.yaml に status として反映し、ヒューマンフレンドリーな進捗レポートを生成

## 実行手順
```yaml
- trigger: "(進捗レビュー|RefineNow|ReFocus)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/dev_tasks.yaml','**/dev_tasks_order.md','**/progress_report.md','**/09_実装__チケット実行と検証/check_*.md']))}}"]
      instructions: |
        直近の進捗から、DONE/IN_PROGRESS/TODOの概要と、推奨の残タスク順（上位5件）を要点だけで出力してください。
      store_as: "auto_refine_hints"
    - name: "prefill_suggestions"
      action: "display"
      content: |
        🔎 進捗スナップショット（自動）
        {{auto_refine_hints}}
    # 1) コンテキスト読み込み
    - name: "show_inputs"
      action: "display"
      content: |
        🔎 読み込み対象（存在すれば使用）
        - `Flow/{{today}}/{{flow_dir}}/07_実装__開発タスク分解/dev_tasks.yaml`
        - `Flow/{{today}}/{{flow_dir}}/07_実装__開発タスク分解/dev_tasks_order.md`
        - `Flow/{{today}}/{{flow_dir}}/07_実装__開発タスク分解/dev_runbook.md`
        - `Flow/{{today}}/{{flow_dir}}/04_仮説駆動__ストーリーマップ/story_map.yaml`
        - `Flow/{{today}}/{{flow_dir}}/05_仮説駆動__UIワイヤーフレーム/screen_map.yaml`
        - `Flow/{{today}}/{{flow_dir}}/06_設計__Drawioスクリーン生成/drawio/`（任意）
        - `Flow/{{today}}/{{flow_dir}}/05_仮説駆動__UIワイヤーフレーム/screen_flow.yaml`
    
    # 2) 進捗の自己申告（柔らかく）
    - name: "collect_progress_mark"
      action: "ask_questions_with_template"
      template: |
        === 進捗チェック ===
        1) 完了タスクID（カンマ区切り。例: T_COMMON_ENV, T_ST2_MODE）
        →
        2) 実行中タスクID（任意）
        →
        3) ブロッカー（任意: 技術/時間/理解）
        →
        =====================================
    
    - name: "wait_progress"
      action: "wait_for_all_answers"
    
    # 3) 再認識サマリの提示（何をすべきか）
    - name: "display_refocus_summary"
      action: "display"
      content: |
        🧭 再認識ポイント
        - ストーリー（story_map）とUI（screen_map）が示す到達像を再確認
        - 共通タスク（Common）が先、次に依存順でStories、最後にNon-Functional
        - 受け入れ基準（AC）と期待ログ（console）の観点を常に併記
    
    # 4) 自動提案（残タスク順序の再構成）
    - name: "propose_refined_order"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/10_進捗レビュー__タスクリファイン/dev_tasks_order_refined.md"
      content: |
        # 提案: 残タスクの実行順（進捗反映）
        {{refined_order}}
        
        ルール:
        - 未完了のCommonを最優先
        - 各Storyは依存を解決しつつ、価値が高い順で
        - NFRは主要UIが成立後に着手
    
    # 5) ステータス反映プラン（DONE/IN_PROGRESS/TODO）
    - name: "display_status_plan"
      action: "display"
      content: |
        🗂 ステータス反映ポリシー
        - 完了（DONE）: 入力で申告した完了タスクID
        - 実行中（IN_PROGRESS）: 入力で申告した実行中タスクID
        - 未着手（TODO）: 上記以外
        dev_tasks.yaml の各タスク要素に `status:` を追加/更新します。
    
    # 6) タスク内容のリファイン入力（任意修正）
    - name: "collect_task_refinements"
      action: "ask_questions_with_template"
      template: |
        === タスク内容のリファイン（任意）===
        1) 変更したいタスクID
        →
        2) 変更内容（title/depends/test_points/estimate_h/status など）
        →
        3) 新規に追加したいタスク（任意: id/title/depends/estimate_h/status）
        →
        =====================================
    
    - name: "confirm_generation"
      action: "confirm"
      message: "進捗反映＋リファイン内容で dev_tasks.yaml / dev_tasks_order.md / dev_runbook.md / progress_report.md を再生成します。よろしいですか？"
    
    # 7) 成成果物の再出力（status付き）
    - name: "write_dev_tasks_yaml"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/07_実装__開発タスク分解/dev_tasks.yaml"
      content: |
        # status欄（DONE/IN_PROGRESS/TODO）を各タスクへ反映済み
        {{dev_tasks_after_refine_and_status_yaml}}
    
    - name: "write_dev_tasks_order"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/07_実装__開発タスク分解/dev_tasks_order.md"
      content: |
        {{final_order_markdown}}
    
    - name: "write_runbook"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/07_実装__開発タスク分解/dev_runbook.md"
      content: |
        # LLM実装Runbook（更新版）
        1) Commonを最優先（未完了分）
        2) 各STは依存順に、価値が高いものから
        3) 受け入れ基準と期待ログ（console）を毎タスクで確認
        4) 完了後は `09_実装__チケット実行と検証` でチェック記録
        5) 必要に応じこの `進捗レビュー` を再実行
    
    # 8) 進捗レポート（人間向け）
    - name: "write_progress_report"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/10_進捗レビュー__タスクリファイン/progress_report.md"
      content: |
        # 進捗レポート（{{today}}）
        
        ## サマリ
        - 完了（DONE）数: {{done_count}}
        - 実行中（IN_PROGRESS）数: {{wip_count}}
        - 未着手（TODO）数: {{todo_count}}
        - 合計タスク数: {{total_count}}
        - 進捗率: {{progress_rate_pct}}%
        - 推奨次タスク（上位3）: {{next_top3}}
        
        ## 詳細一覧
        ### DONE
        {{done_list}}
        
        ### IN_PROGRESS
        {{wip_list}}
        
        ### TODO（優先順）
        {{todo_ordered_list}}
        
        ## ブロッカー/メモ
        {{blockers_text}}
        
        ## 参照
        - ../07_実装__開発タスク分解/dev_tasks.yaml（status反映済）
        - ../07_実装__開発タスク分解/dev_tasks_order.md（最終順序）
        - ../07_実装__開発タスク分解/dev_runbook.md（更新版）
        - ../04_仮説駆動__ストーリーマップ/story_map.yaml / ../05_仮説駆動__UIワイヤーフレーム/screen_map.yaml / screen_flow.yaml
    
    # 9) 次アクション提案（提出/発表 もしくは 次チケット）
    - name: "write_next_action"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/10_進捗レビュー__タスクリファイン/next_action.md"
      content: |
        # 次アクション提案
        
        - 進捗率: {{progress_rate_pct}}%（{{done_count}} / {{total_count}}）
        - 残タスク: {{todo_count}}（実行中: {{wip_count}}）
        
        ## 推奨
        {{#progress_rate_pct >= 80}}
        ほぼ完了と判断。以下を実行してください：
        1. 発表: 「発表_Marpスライド生成」または `@11_発表__Marpスライド生成`
        2. 提出: 「提出_成果物パッケージング」または `@12_提出__成果物パッケージング`
        
        参考: submitフォルダ名は `submit_{{today}}` を推奨
        {{/progress_rate_pct >= 80}}
        {{^progress_rate_pct >= 80}}
        まだ実装継続を推奨。次に着手すべきチケット：
        - 推奨: {{next_top1}}
        実行コマンド: `@08_実装__チケット開始` を実行し、上記 `task_id` を指定してください。
        {{/progress_rate_pct >= 80}}
    
    - name: "display_next_action"
      action: "display"
      content: |
        📣 進捗率: {{progress_rate_pct}}%（DONE {{done_count}} / {{total_count}}）
        
        {{#progress_rate_pct >= 80}}
        ✅ ほぼ完了です。次は `11_発表__Marpスライド生成` → `12_提出__成果物パッケージング` の順で仕上げましょう。
        {{/progress_rate_pct >= 80}}
        {{^progress_rate_pct >= 80}}
        ▶ 次チケットに着手: {{next_top1}}（`@08_実装__チケット開始`で `task_id` 指定）
        {{/progress_rate_pct >= 80}}
    
    - name: "notify"
      action: "display"
      content: |
        ✅ 再生成しました：
        - ../07_実装__開発タスク分解/dev_tasks.yaml（status反映） / dev_tasks_order.md / dev_runbook.md
        - 10_進捗レビュー__タスクリファイン/progress_report.md（サマリ＋一覧）
        - 10_進捗レビュー__タスクリファイン/next_action.md（次アクション提案）
        - 提案順序: 10_進捗レビュー__タスクリファイン/dev_tasks_order_refined.md
        次は `08_実装__チケット開始` に戻って継続実行してください。
```

## メモ
- 既存ファイルが無い場合は、ある範囲で空を許容しつつ入力ベースで再生成します。
- 進捗記録は `../07_実装__開発タスク分解/dev_tasks.yaml` へ `status` を追記して保持します（DONE/IN_PROGRESS/TODO）。
