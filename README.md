# Midas-log

midas（日本株の自動運用ソフト。開発は非公開リポジトリ）のフォワードテストにおける、**シグナルの事前コミット記録**。

このリポジトリには**ハッシュ値しか置かない**。目的はただ一つ、「そのシグナルは、結果が出る前に確定していた」ことを第三者が後から検証できるようにすることにある。

## なぜハッシュだけなのか

運用戦略の成績は、後から出す限りいくらでも良く見せられる。都合の良い日だけ数える、結果を見てから「それは狙っていた」と言う、負けた戦略を無かったことにする——どれも外からは見分けがつかない。

そこで毎営業日、シグナルが確定した時点で**その内容の SHA-256 だけ**をこのリポジトリにコミットする。中身は公開しないが、ハッシュは内容に対する変更不能な約束になる。後日、非公開側の記録を提示されたとき、誰でも突合して「この日のシグナルは確かにこれだった」ことを確認できる。

コミットの日時は Git が刻む。判定に使うラベルは執行から5営業日後（T+6の寄付）まで確定しないので、**T 日付のコミットはシグナルが自らの結果に先行していたことの証拠**になる。

## 置いてあるもの / 置いていないもの

```
signals/{YYYY}/{YYYY-MM-DD}.json
```

```json
{
  "spec": "midas-forward-v1",
  "date": "2026-08-07",
  "entries": [
    { "strategy_id": "baseline_reversal_top10", "sha256": "96ce7a26…" }
  ]
}
```

置いてあるのは **日付・戦略ID・SHA-256 のみ**。

銘柄コード・銘柄名・シグナル値・ウェイト・保有数量・価格・売買履歴・生データは**一切含まない**。非公開側にも、公開直前にこの構造を再検査して逸脱を例外にする実装上のガードがある。

`baseline_` で始まる戦略IDは比較用のベースライン（ベンチマークETF買い持ち、ユニバース等ウェイト、単純な横断リバーサル）で、常時並走している。

## 正規化仕様（`midas-forward-v1`）

ハッシュ対象のバイト列は完全に固定されている。1日 × 1戦略で1文書：

```
midas-forward-v1
date=2026-08-07
strategy=baseline_reversal_top10
code,signal,weight
1301,-0.0123456789,0.0000000000
1332,0.0034567890,0.1000000000
```

- 文字コード UTF-8、改行 LF（`\n`）、最終行の後にも改行。BOM・CR・余白なし
- 1行目は仕様ID、2〜3行目は `key=value`、4行目は列見出し `code,signal,weight` の固定文字列
- データ行はその日その戦略の**記録済み横断面すべて**（保有銘柄だけではない）。`code` の ASCII バイト昇順
- 区切りは単一のカンマ。引用符・空白は入れない
- `signal` と `weight` は**固定小数点10桁**（IEEE-754 double に対する偶数丸め。Python の `format(x, '.10f')` と同一）。`-0` は `0.0000000000` に正規化。非有限値は `nan` / `inf` / `-inf`
- 保有していない銘柄の `weight` は `0.0000000000`

この UTF-8 バイト列の SHA-256（小文字16進）が `sha256` の値。

## 検証手順

非公開側の記録（`data/forward/{YYYY}/{YYYY-MM-DD}.parquet`、列は `code, signal, weight`）を提示された場合：

1. 対象日の `signals/{YYYY}/{YYYY-MM-DD}.json` を取得する
2. `git log -- signals/{YYYY}/{YYYY-MM-DD}.json` でコミット日時を確認する（＝「いつ」の証拠）
3. 提示された記録を上の仕様どおりに再直列化し、SHA-256 を計算する
4. `entries` の当該 `strategy_id` の `sha256` と一致するか照合する（＝「何を」の証拠）

参考実装（提示された記録が CSV でも parquet でも、列が揃っていれば同じ）：

```python
import hashlib

def canonical_text(day, strategy_id, rows):   # rows: (code, signal, weight) の列
    def fmt(x):
        if x != x:
            return "nan"
        s = format(float(x), ".10f")
        return s[1:] if s.startswith("-") and float(s) == 0.0 else s

    lines = ["midas-forward-v1", f"date={day}", f"strategy={strategy_id}", "code,signal,weight"]
    lines += [f"{c},{fmt(s)},{fmt(w)}" for c, s, w in sorted(rows, key=lambda r: str(r[0]))]
    return "\n".join(lines) + "\n"

digest = hashlib.sha256(canonical_text("2026-08-07", "baseline_reversal_top10", rows).encode()).hexdigest()
```

## この記録が証明すること／しないこと

- **する**：あるシグナルが、その結果が判明する前に確定していたこと。事後の改竄・後出し・戦略の隠蔽が起きていないこと
- **しない**：戦略が有効であること。ハッシュは順序の証拠であって、成績の証拠ではない。成績の判定は非公開側で事前登録された基準に従って別途行われる
- 公開開始日より前の日付に遡って追加されたコミットは、当然ながら先行性の証拠にならない。`git log` の日時が全て

## ライセンス・関係

このリポジトリは公開。midas 本体（コード・データ・戦略）は非公開で、成果物としてここに出るのはハッシュのみ。シグナルの販売・配信は行っていない。
