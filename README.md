# 1Password Custom Field Autofill Matching Lab

1Password のカスタムフィールドが、DOM 上のどの識別情報（`id` / `name` / `placeholder` / `label` / ARIA 属性 / 周辺テキストなど）と一致したときに Autofill されるのかを、ブラックボックスで検証するための単一 HTML ページです。

ビルド不要・依存パッケージなしで、[index.html](index.html) をブラウザで開くだけで動作します。

## 検証の仕組み

- ページ内に、識別情報の与え方だけを変えた入力フィールドを多数配置しています
- 1Password 側にカスタムフィールド **`OP_TEST_TOKEN`** = **`FILLED_BY_1PASSWORD`** を用意し、このページで Autofill を実行します
- 各フィールドについて、以下のいずれかを満たしたものを「一致した（ヒット）」とみなします
  - 値が `FILLED_BY_1PASSWORD` になっている
  - 1Password が付与する `data-com-onepassword-filled` 属性が付いている
- ヒットしたフィールドは枠線でハイライトされ、各行の右側に属性のスナップショットが JSON で表示されます

> [!IMPORTANT]
> 実測では、1 回の Autofill で埋まるのは**カスタムフィールド 1 つにつき 1 フィールドのみ**でした。
> つまり「埋まらなかった = マッチしなかった」とは断定できません。マッチはしたが選ばれなかっただけの可能性があります。
> 属性ごとの判定を確実に取るには、1 ケースだけを残して他を取り除いた状態で 1 回ずつ試す必要があります（[実験モード](#実験モードurl-パラメータ)の `?only=` を使ってください）。

## 使い方

### 1. 1Password 側の準備

1. 任意のログインアイテム（このページのドメイン／ファイルに紐づくもの）を開く
2. カスタムフィールドを追加する
   - 種類: **Text**
   - ラベル: `OP_TEST_TOKEN`
   - 値: `FILLED_BY_1PASSWORD`
3. 保存する

> [!NOTE]
> **カスタムフィールドの種類は充填先を決めます。** input の `type` に対応する種別だけが充填されるため、`type=email` を試すならメール種別、`type=password` ならパスワード種別が必要です（[docs/findings.md の実験 4](docs/findings.md) 参照）。
> 同名で種別違いのカスタムフィールドを同時に設定しても競合しません。結果を記録する際は**使用した種別を必ず併記**してください。

### 2. ページを開く

```bash
open index.html
```

`file://` で開いた場合に拡張機能が動作しないときは、ローカルサーバー経由で開いてください。

```bash
python3 -m http.server 8000
# → http://localhost:8000/
```

### 3. Autofill して結果を見る

1. 1Password の拡張機能から Autofill を実行する
2. 「結果を再スキャン」を押す（属性変化は MutationObserver でも自動検知されます）
3. ページ上部のサマリと、各行の JSON でどのフィールドがヒットしたかを確認する

## 実験モード（URL パラメータ）

「1 回の Autofill で 1 フィールドしか埋まらない」という制約を回避し、選定基準を切り分けるためのモードです。
1Password はフィールドの解析結果をキャッシュするため、読み込み後に JS で DOM を操作しても認識が更新されない可能性があります。そのため、これらは URL パラメータとして**読み込み時点**で適用されます。

| パラメータ | 動作 | 用途 |
| --- | --- | --- |
| `?only=<ケースID>` | 指定ケース以外の `.case` を DOM から**削除**し、input を 1 つだけ残す | 属性ごとのマッチ有無を単独で判定する |
| `?reverse=1` | 各セクション内のケース順を読み込み時点で反転する | DOM 順の影響の検証。反転時は `7-A` がページ最後の input になる |

併用可能です（例: `?only=7-A&reverse=1`）。ケース ID は `0-A` 〜 `7-C` で、ページ上部のモードパネルにある「1ケースだけを残して開く」から全 31 件のリンクを辿れます。存在しない ID を指定した場合は全ケース表示にフォールバックし、警告を表示します。

```
index.html?only=1-C          # placeholder一致のケースだけを残す
index.html?only=2-A          # label for のケースだけを残す
index.html?reverse=1         # 全ケースを読み込み時点で逆順にする
```

> [!TIP]
> 単独モードで 1 ケースずつ試すと、「マッチする属性の一覧」が確定します。これが分かってから `?reverse=1` で位置の影響を見ると、結果を解釈しやすくなります。

## ツールバー

| ボタン | 動作 |
| --- | --- |
| 結果を再スキャン | 全フィールドを再判定して表示を更新する |
| 入力値をクリア | 値と `data-com-onepassword-filled` を消して初期状態に戻す |
| DOM順を反転（読み込み後） | セクション内の行順を逆にする。**読み込み後**の並べ替えなので、1Password が読み込み時に構築した索引には反映されない可能性がある。tie-break の検証には `?reverse=1` を使うこと |
| 結果JSONをコピー | モード情報・UserAgent・全フィールドの属性スナップショットをクリップボードにコピーする |

## テストケース一覧

| セクション | 検証内容 |
| --- | --- |
| 0. Baseline / DOM順 | 識別情報を持たない通常 input。誤爆や DOM 順だけでの充填が起きないかの基準 |
| 1. 直接属性 | `id` / `name` / `placeholder` / `title` の一致 |
| 2. label | `label for` / label 内包 / 関連なし label / 壊れた `for` / DOM 上離れた `for` |
| 3. ARIA | `aria-label` / `aria-labelledby` / `aria-describedby` |
| 4. 周辺テキスト / Tooltip | 直前の `span` / `div`、CSS ツールチップ、`hidden` テキスト |
| 5. 要素 / input type | `textarea` と `type=text/search/email/password` の差 |
| 6. form / autocomplete | form 内外の差、`autocomplete=off` / `autocomplete=username` |
| 7. 完全重複候補 | 同一の識別情報を持つ 3 フィールド。tie-break が DOM 順かの確認 |

## 結果の記録

「結果JSONをコピー」で取得できる JSON は次の形です。実行モードと UserAgent が含まれるため、条件込みでそのまま保存できます。

```jsonc
{
  "mode": { "only": "1-C", "reverseAtLoad": false },
  "userAgent": "...",
  "fields": [
    {
      "case": "1-C", "tag": "input", "type": "text",
      "id": "placeholder-test", "name": "placeholder_test",
      "placeholder": "OP_TEST_TOKEN", "labels": [],
      "value": "FILLED_BY_1PASSWORD",
      "onePasswordFilled": "light", "domIndex": 0
      // title / ariaLabel / ariaLabelledby / ariaDescribedby / autocomplete も含む
    }
  ]
}
```

JSON に含まれないため、別途記録しておくと有用な条件:

- **使用したカスタムフィールドの種別**（テキスト / メール / パスワード など）— 充填先の `type` が変わるため必須
- 1Password 拡張機能のバージョン
- OS
- Autofill の実行方法（ショートカット / 拡張機能のポップアップ / インラインアイコン）

## 検証結果の要点

詳細は [docs/findings.md](docs/findings.md)、外部ソースとの照合は [docs/sources.md](docs/sources.md) を参照してください。

### 確定していること

- **input の `type` は「どの種別のカスタムフィールドを受け付けるか」を決める。** `type=text` にはテキスト種別、`type=email` にはメール種別、`type=password` にはパスワード種別が充填される。対応しない種別は充填されない
- **同名で種別違いのカスタムフィールドが共存しても競合しない。** テキスト / メール / パスワードの `OP_TEST_TOKEN` を同時に設定しても、各 input の `type` に対応する種別だけが選ばれる
- `type=search` は充填されない（対応する種別が存在しない）。`textarea` はテキスト種別では充填されない（住所種別は未検証）
- **一致に使われる情報源**: `id` / `name` / `placeholder` / `title` / `aria-label` / `aria-labelledby` / `label`（正式な関連付けの有無を問わない）/ **input より前方の可視テキスト**
- **不可視テキストは無視される**（検証したのは `display:none` と `hidden` 属性の 2 通り）
- **`aria-describedby` はラベル情報源として使われない**
- **`autocomplete=off` は充填を防げない**。form の内外も無関係
- **1 回の Autofill で埋まるのはカスタムフィールド 1 つにつき 1 フィールドのみ**
- 識別情報を持たないフィールドへの誤爆はない

### 未解決の論点

| 問い | 状態 |
| --- | --- |
| 住所種別は `textarea` に充填されるか | 未検証。住所種別の `OP_TEST_TOKEN` を作り `?only=5-A` を実行すれば、コード変更なしに検証できる |
| URL / 日付 / 電話 / サインイン 種別と `type=url` / `date` / `tel` の対応 | 未検証。対応する input のケースが存在しない |
| 選定は「スコア最大」か「単に最後の一致要素」か | 現構成では両説が同一の予測を出すため分離不可。高スコア候補を**前方**に置くケースが必要 |
| 後方の可視テキストを拾うか | 未検証。該当ケースが存在しない |
| `id` / `name` の辞書順が影響するか | `?reverse=1` で検証可能 |
| 完全一致だけでなく**部分一致**でも充填されるか | 未検証（外部記事は部分一致と報告） |
| `data-1p-ignore` は実際に充填を止めるか | 未検証 |

> [!WARNING]
> 1Password にはサイト個別の **recipe**（個別ルール）が存在し、挙動が上書きされ得ます。上記はいずれも **recipe のないページでの既定挙動**であり、任意のサイトにそのまま一般化はできません。詳細は [docs/sources.md](docs/sources.md#外部ソースから判明した重要な但し書き) を参照してください。

## 注意

- **本プロジェクトは非公式であり、AgileBits / 1Password とは一切関係ありません。** 1Password は AgileBits Inc. の商標です
- 本リポジトリは外形的な挙動を観察するためのページであり、公式な仕様を示すものではありません。挙動はバージョンによって変わり得ます
- 記録済みの結果は **1Password for Mac 8.12.33 / Chrome 152 / macOS 15.5** での単発の観測です（[docs/findings.md](docs/findings.md#検証環境) 参照）
- **サイト個別の recipe により挙動が上書きされる場合があります**（[docs/sources.md](docs/sources.md#外部ソースから判明した重要な但し書き) 参照）。本ラボの結果は recipe のないページでの既定挙動です
- `FILLED_BY_1PASSWORD` はダミー値です。実際の認証情報をカスタムフィールドに入れて検証しないでください

## ライセンス

[MIT License](LICENSE)

## 構成

```
.
├── index.html          # 検証ページ本体（HTML / CSS / JS すべて内包）
├── README.md           # 概要・使い方・結果の要点
├── LICENSE             # MIT
└── docs/
    ├── findings.md     # 検証環境、観測結果の詳細、仮説、次に必要な実験
    └── sources.md      # 外部ソースとの突き合わせ（裏付け / 矛盾）
```
