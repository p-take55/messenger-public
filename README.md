# messenger

aachat とローカル実行基盤をつないで、Slack、Chatwork、Gmail などの業務メッセージを agent workflow に載せるエージェントです。

`messenger` は provider webhook を直接待ち受ける常駐 agent ではありません。ローカルマシン側の `reply-drafter` CLI / service が message を受信し、ローカル SQLite に workflow state を保存します。`messenger` は必要なときだけ aachat session として起動し、未処理 message の triage、返信案作成、asks document 作成、承認後送信の指示を担当します。

## 役割

- ローカルランナーの初回設定を案内する
- ローカル SQLite に溜まった未処理 message を確認する
- 返信すべき message を判定する
- Slack、Chatwork、Gmail の文脈に合う返信案を作る
- aachat docs に asks document を作り、人間に承認を求める
- asks 回答後、ローカルランナーに承認済み送信を任せる
- 久々起動時に backlog、stuck state、backfill 要否を確認する

## 構成

```text
messenger/
├── README.md
├── identity.md
├── environment.yaml
├── knowledge/
│   ├── architecture.md
│   ├── local-runner-protocol.md
│   ├── provider-adapters.md
│   └── reply-policy.md
├── memory/
└── .agents/
    └── skills/
        ├── first-run-setup/
        ├── check-inbox/
        ├── draft-reply/
        ├── handle-ask-answer/
        └── recover-backlog/
```

## Runtime model

```text
Provider
  -> Provider SDK Adapter
  -> Local Runner
  -> Local SQLite
  -> messenger agent session
  -> aachat Docs / Asks
  -> Human Decision
  -> Local Runner
  -> Provider
```

Provider SDK や API client はローカルランナー内の Provider Adapter に閉じ込めます。`messenger` agent は Slack / Gmail / Chatwork SDK の細部を直接扱わず、ローカルランナーが返す normalized message と workflow state を見る前提です。

## 期待するローカル CLI

最初の実装では、ローカルマシンに `reply-drafter` CLI が入っている前提にします。

```bash
reply-drafter init
reply-drafter start
reply-drafter stop
reply-drafter status --json
reply-drafter doctor --json
reply-drafter catch-up --json
reply-drafter process --json
reply-drafter send-approved --json
```

CLI が未インストールなら、`messenger` はインストールと設定の案内を行います。任意コマンドを推測で実行せず、安定した CLI surface だけを使います。

この repo 自体にも、最小のローカルランナー CLI stub を同梱しています。開発中は次のようにインストールできます。

```bash
pipx install git+https://github.com/p-take55/messenger-public.git
```

ローカルで直接試す場合:

```bash
python -m messenger_local_runner.cli status --json
python -m messenger_local_runner.cli init --json
```

この stub は SQLite schema 作成と status / doctor までを担当します。Slack、Gmail、Chatwork の Provider Adapter、background service、実送信はまだ未実装です。

## 安全境界

- 外部 provider への送信は、人間の asks 回答後だけ行う
- 送信実行はローカルランナーに任せる
- token、secret、API key の値を chat、docs、SQLite に出さない
- ローカル SQLite は workflow state の正本だが、secret store ではない
- session workspace の一時ファイルを workflow 正本にしない
