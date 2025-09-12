# 01 仮説駆動: ペルソナ作成（AIPMハッカソン）

## 最初に質問（実行前に回答してください）
- 誰のためのアプリ（例: TODOアプリ）を作りますか？（必須）
- 年齢・性別・職業は？（必須）
- その人の1日の流れは？（必須）
- アプリに関する悩みは？（必須）
- 大切にしている価値観は？（任意）

> ヒント: `/aipm_hackathon/persona_candidates_100.md` を見ながら発想してOK！

## 目的
アプリのターゲットユーザー（例: TODOアプリ）を具体化し、「仮説」として後で検証・修正できる状態に落とし込みます。

## 実行手順
```yaml
- trigger: "(仮説駆動_ペルソナ作成|HypothesisPersona)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}", "{{read_files(find_files(patterns=['Flow/**/*.md','Flow/**/*.mdx','**/submit_*/README.md']))}}"]
      instructions: |
        直近スレッドと同スレッドの成果物から、以下キーを推定（なければ空）。
        keys: persona_who, age_gender, occupation, daily_routine, app_problem, values
        出力はJSONのオブジェクト1件のみ。
      store_as: "prefill"

    - name: "ingest_sense_outputs"
      action: "analyze"
      data: [
        "{{read_files(find_files(patterns=['**/1_sense/02_sense__顧客調査/sense_customer_research.md']))}}",
        "{{read_files(find_files(patterns=['**/1_sense/05_sense__インタビュー分析（個別）/draft_interview_analysis_*.md']))}}",
        "{{read_files(find_files(patterns=['**/1_sense/06_sense__リサーチサマリー（全体）/draft_research_summary.md']))}}"
      ]
      instructions: |
        顧客調査・インタビュー分析・リサーチサマリーから、ペルソナの補助候補値を抽出してJSONで返してください。
        keys: persona_who_hint, age_gender_hint, occupation_hint, daily_routine_hint, app_problem_hint, values_hint
      store_as: "sense_hints"
    - name: "show_persona_examples"
      action: "display"
      content: |
        💡 ペルソナの例（参考）
        - 働くママ（35）: 家事と仕事のタスクが混在
        - 大学生（20）: バイトや課題でスケジュール混乱
        - フリーランス（28）: 締切管理と優先度が苦手
        - シニア（68）: 小さい文字が見づらい
        
        さらに → `/aipm_hackathon/persona_candidates_100.md`
    
    - name: "collect_persona_info"
      action: "ask_questions_with_template"
      template: |
        === ペルソナ情報入力テンプレート ===
        1. 誰のためのアプリ（例: TODOアプリ）？
        （候補: {{prefill.persona_who}} / Sense: {{sense_hints.persona_who_hint}}）
        → 【あなたの回答】：
        
        2. 年齢・性別・職業は？
        （候補: {{prefill.age_gender}} / {{prefill.occupation}} / Sense: {{sense_hints.age_gender_hint}} / {{sense_hints.occupation_hint}}）
        → 【あなたの回答】：
        
        3. 1日の流れ（時系列で3-6項目）
        （候補: {{prefill.daily_routine}} / Sense: {{sense_hints.daily_routine_hint}}）
        → 【あなたの回答】：
        
        4. アプリに関する悩み
        （候補: {{prefill.app_problem}} / Sense: {{sense_hints.app_problem_hint}}）
        → 【あなたの回答】：
        
        5. 大切にしている価値観（任意）
        （候補: {{prefill.values}} / Sense: {{sense_hints.values_hint}}）
        → 【あなたの回答】：
        =====================================
    
    - name: "wait_inputs"
      action: "wait_for_all_answers"
    
    - name: "refine_brief"
      action: "interactive_dialog"
      message: |
        いいですね！
        ペルソナの悩みを1行で要約してください（仮説として）。
        例）「選択肢が多すぎて、最初の一歩が出ない」
    
    - name: "confirm_generate"
      action: "confirm"
      message: "収集した内容で全成果物（persona.md / experience_map.yaml / mermaid）を生成します。よろしいですか？"
    
    - name: "generate_persona"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/01_仮説駆動__ペルソナ作成/persona_todo.md"
      content: |
        # アプリ ペルソナ仮説（例: TODOアプリ）
        
        ## 基本情報
        - 名前（任意）: {{generated_name}}
        - 年齢・性別: {{age_gender}}
        - 職業: {{occupation}}
        
        ## 1日の流れ
        {{daily_routine}}
        
        ## アプリに関する課題（仮説）
        - 要約: {{problem_brief}}
        - 具体例: {{specific_problems}}
        
        ## 価値観・重視
        {{values}}
        
        ## 注意
        - 本文書は仮説です。検証前提で更新します。
        - 思い込みの可能性: {{assumptions}}
    
    - name: "export_experience_map_yaml"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/01_仮説駆動__ペルソナ作成/experience_map.yaml"
      content: |
        persona:
          id: P001
          name: {{generated_name}}
          profile:
            age_gender: {{age_gender}}
            occupation: {{occupation}}
          behaviors: {{daily_routine | to_list}}
          values: {{values}}
        experience_map:
          phases:
            - id: PH1
              name: "買い物前の準備"
              actions:
                - id: A101
                  label: "例: 連絡帳を書く"
                  notes: []
            - id: PH2
              name: "レジ/作業中"
              actions:
                - id: A201
                  label: "例: その場で5分タスクに着手"
                  notes: []
            - id: PH3
              name: "完了後の見直し"
              actions:
                - id: A301
                  label: "例: 今日の3枠を確認"
                  notes: []
        numbering_policy:
          - prefix: P=Persona, PH=Phase, A=Action
          - 以後の課題(PR)/解決策(SL)/ストーリー(ST)と連番で紐付けます
    
    - name: "export_experience_map_mermaid"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/01_仮説駆動__ペルソナ作成/experience_map_mermaid.md"
      content: |
        ```mermaid
        flowchart LR
          P001[ペルソナ: {{generated_name}}]
          P001 --> PH1[PH1 買い物前の準備]
          P001 --> PH2[PH2 レジ/作業中]
          P001 --> PH3[PH3 完了後の見直し]
          PH1 --> A101[A101 行動]
          PH2 --> A201[A201 行動]
          PH3 --> A301[A301 行動]
        ```
    
    - name: "notify_completion"
      action: "display"
      content: |
        ✅ 生成しました：
        - 01_仮説駆動__ペルソナ作成/persona_todo.md
        - 01_仮説駆動__ペルソナ作成/experience_map.yaml
        - 01_仮説駆動__ペルソナ作成/experience_map_mermaid.md
```

## 次のコマンド
→ `仮説駆動_課題定義` で課題を深掘りします
