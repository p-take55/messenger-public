---
name: handle-ask-answer
description: >
  aachat asks document で人間が回答した後、SQLite decision への同期とローカルランナーによる送信 / close を扱う。
---

# handle-ask-answer

asks 回答後に使う。目的は、人間の判断を durable decision に反映し、送信または close をローカルランナーへ任せること。

## 原則

- agent 自身が provider へ直接送信しない。
- `reply-drafter send-approved --json` など、安定したローカルランナー CLI surface を使う。
- 同じ reply_request を二重送信しない。
- 送信結果を aachat docs と SQLite state に反映する。

## 手順

1. asks document の回答を確認する。
2. 採用、編集して採用、不要として閉じる、再生成のどれかを判定する。
3. ローカルランナーに decision sync / send-approved を依頼する。
4. 送信成功なら asks document を sent にする。
5. close なら asks document を closed にする。
6. error なら error 内容を secret を含まない形で報告し、needs_attention にする。

## 注意

credential 値、provider token、raw secret を chat や docs に出さない。
