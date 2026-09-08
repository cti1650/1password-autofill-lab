# 外部ソースとの突き合わせ

[← README に戻る](../README.md) ｜ [観測結果の詳細 → findings.md](findings.md)

本ラボの観測結果を、公式ドキュメント・公式スタッフ発言・同種の検証記事と照合した結果です。
ソースごとに**裏付けポイント**と**矛盾・相違ポイント**を明記しています。

## 照合サマリ

| 本ラボの観測 | 外部ソース | 判定 |
| --- | --- | :---: |
| `type=search` は充填されない（5-C、**Text 種別のカスタムフィールド使用時**） | 1Password 社員 1P_Dave（2025-04-15）「1Password won't fill into fields marked as _type="search"_ if our team hasn't created a specific recipe for a website」 | 一致 |
| `autocomplete=off` でも充填される（6-C） | 公式開発者ドキュメントはオプトアウト手段として `data-1p-ignore` / `data-op-ignore` を案内。`autocomplete=off` は無効化手段として記載されていない | 一致 |
| `id` / `name` / `label` で一致（1-A / 1-B / 2 系） | mizdra 氏の検証記事（2021-10-05）が同じ 3 つを挙げている。公式コミュニティでもスタッフが HTML ID による方法を案内（2023-01-12） | 一致 |
| `aria-label` / `aria-labelledby` が有効（3-A / 3-B） | 公式開発者ドキュメント「When examining a page, 1Password can take advantage of accessibility cues to locate fields」、ARIA 属性でのアノテーションを推奨 | 一致 |
| `textarea` は充填されない（5-A、**Text 種別のカスタムフィールド使用時**） | 公式コミュニティに「Autocomplete ignores textarea fields」というスレッドが存在し、textarea を text input に変えると動くとの報告（**リンク切れのため本文未確認**） | 概ね一致 |
| `placeholder` が有効（1-C） | 公式・mizdra 氏いずれも言及なし | **本ラボで新規確認** |
| 前方の可視テキストが有効（4-A / 4-B）、不可視テキストは無効（4-C / 4-D） | 該当する記述を発見できず | **本ラボで新規確認** |
| `aria-describedby` は使われない（3-C） | 該当する記述を発見できず | **本ラボで新規確認** |

## 外部ソースから判明した重要な但し書き

**1. サイト個別の「recipe」が存在する**

1P_Dave の回答にある `recipe` は、1Password が特定サイト向けに用意する個別ルールを指します。つまり**充填挙動はサイトによって上書きされ得ます**。本ラボの結果は「recipe が存在しないサイトでの既定挙動」として解釈すべきで、任意のサイトにそのまま一般化はできません。

**2. フィールド解析結果はキャッシュされる（走査キャッシュ仮説の裏付け）**

1Password 社員 1P_Dave がコミュニティで次のように明言しています（2025-06-27）。

> When a user clicks into a field, 1Password will analyze the page and note whether **data-1P-ignore** has been used. After this initial analysis, **the field type is cached and isn't analyzed again**.

これは本ラボで「読み込み後に JS で並べ替えても結果が変わらなかった」ことの説明になります。

なお公式開発者ドキュメントにキャッシュの記述はありません（「Don't dynamically add or remove fields from the DOM」という推奨があるのみ）。この情報の出典はコミュニティでのスタッフ発言です。

**3. 一致は「部分一致」の可能性がある（未検証）**

mizdra 氏の記事は、カスタムフィールド名が `name` / `id` / `label` の値に**部分一致**する場合に充填されると報告しています。本ラボは完全一致しか試していないため、部分一致の可否は未検証です。ただし同記事は「undocumented な機能だったので、正確な仕様は分かりません」と留保を付けています。

**4. 型ゲートは「カスタムフィールドの種別」との対応である可能性が高い**

1Password のカスタムフィールドには **テキスト / URL / メール / 住所 / 日付 / ワンタイムパスワード / パスワード / 電話 / サインイン** の種別があります。本ラボは **Text 種別しか使用していない**ため、5 系で得た「`input[type=text]` 以外は充填されない」という結果は Text 種別に限った話です。

メール種別なら `input[type=email]`、パスワード種別なら `input[type=password]` に充填される可能性が高く、**未検証**です。1P_Dave の `type=search` に関する発言も「search に対応する種別が存在しないため recipe が必要」と読めば整合します。

**5. 隠し input と隠しテキストは別問題**

公式ドキュメントはパスワード変更フローで `style="display: none;"` の username フィールドを置くことを推奨しており、**隠された input 自体は充填対象になり得ます**。本ラボの 4-C / 4-D は「隠されたテキストを**ラベル情報源**として使うか」を見たもので、論点が異なります。隠し input への充填可否は未検証です。

## 個別ソースの評価

### [Design your website to work best with 1Password（1Password Developer 公式）](https://www.1password.dev/web/compatible-website-design/)

**裏付けポイント**

- 「When examining a page, 1Password can take advantage of accessibility cues to locate fields」および ARIA 属性でのアノテーション推奨 → **3-A / 3-B（`aria-label` / `aria-labelledby`）が有効**だったことと整合
- `<label>` と `for` 属性の使用、`id` / `name` の一意性を推奨 → **1-A / 1-B / 2-A** の結果と整合
- オプトアウト手段として `data-1p-ignore` / `data-op-ignore` を案内。`autocomplete="off"` は無効化手段として一切挙げられていない → **6-C で `autocomplete=off` でも充填された**ことと整合
- 「Don't dynamically add or remove fields from the DOM. Reuse fields and hide them when you don't need them.」という注意書き → 読み込み後の DOM 変更が認識されない前提と整合し、**反転ボタンで結果が変わらなかった**ことと符合

**矛盾・相違ポイント**

- **`placeholder` への言及が一切ない**が、本ラボでは 1-C で有効だった。公式ドキュメントが列挙する識別手段は網羅的ではない
- 近傍テキスト（2-C / 2-D / 4-A / 4-B）についても記述がない。**正式な関連付けがなくても充填される**という本ラボの発見は、公式ドキュメントの記述範囲を超えている
- 公式はパスワード変更フローで `style="display: none;"` の username フィールドを置くことを推奨しており、**隠し要素を扱う前提に見える**。一方 4-C / 4-D では隠しテキストが無視された。ただし前者は「隠された**入力欄**」、後者は「隠された**ラベル用テキスト**」で論点が異なるため、直接の矛盾とは言い切れない（隠し input 自体への充填は本ラボ未検証）
- 充填可能な input type について明示的な記述がない。`type=text` / `type=password` が例示されるのみで、**5 系で判明した型ゲートは文書化されていない**。カスタムフィールドの種別と input `type` の対応関係についても記述がなく、公式情報からは確認できない

### [Request: 1Password support for autofill on input fields with type="search"（1Password Community, 2025-04-15）](https://www.1password.community/discussions/1password/request-1password-support-for-autofill-on-input-fields-with-typesearch/152852)

**裏付けポイント**

- 1Password 社員 1P_Dave「1Password won't fill into fields marked as _type="search"_ if our team hasn't created a specific recipe for a website using that type of field for a login form」 → **5-C（`type=search`）が充填されなかった**ことの公式かつ直接的な裏付け。本ラボの型ゲート説を最も強く支持するソース
- 投稿者 dalfajk は「カスタムフィールドのラベルをおそらく全種類試したが動かなかった（No, it doesn't work. I tried probably all those kinds of labels.）」と報告。対象が `type=search` である以上、ラベルの与え方によらず充填されない。型ゲートの結果と整合する

**矛盾・相違ポイント**

- **`recipe`（サイト個別ルール）の存在**が明示されている。recipe があれば `type=search` でも充填され得るため、型ゲートは絶対的な制限ではない
- 5-C の「無効」は「recipe がないため無効」であり、恒久的な仕様上の制限とは限らない

### [Adding data-1p-ignore dynamically isn't respected（1Password Community, 2025-06-27）](https://www.1password.community/discussions/developers/adding-data-1p-ignore-dynamically-isnt-respected/157845)

**本ラボの走査キャッシュ仮説を裏付ける唯一の一次ソース。**

**裏付けポイント**

- 1Password 社員 1P_Dave「When a user clicks into a field, 1Password will analyze the page and note whether **data-1P-ignore** has been used. After this initial analysis, the field type is cached and isn't analyzed again.」 → **反転ボタン（読み込み後の JS による DOM 操作）で結果が変わらなかった**ことの説明になり、`?reverse=1` を実装した根拠を裏付ける

**矛盾・相違ポイント**

- 発言は **`data-1p-ignore` の検出タイミング**についてのものであり、「充填先の選定結果までキャッシュされるか」「並べ替えを検知しないか」は明言されていない。走査キャッシュ仮説はこの発言からの推論
- 解析の起点が「ユーザーがフィールドをクリックしたとき」とされており、ページ読み込み時に走査されるという前提とは異なる。クリック時点で再解析されるなら読み込み後の並べ替えも反映され得るため、`?reverse=1` が走査キャッシュを回避できるかは未確定

### [1Password のカスタムフィールドを autofill に利用する（mizdra's blog, 2021-10-05）](https://www.mizdra.net/entry/2021/10/05/004744)

**本ラボと最も目的が近い、個人による検証記事。**

**裏付けポイント**

- カスタムフィールド名が入力欄の `name` 属性・`id` 属性・`label` 要素のテキストに一致すると充填される、と報告 → **1-A / 1-B / 2 系**の結果と一致。独立した検証で同じ 3 経路が確認されている
- 「undocumented な機能だったので、正確な仕様は分かりません」と留保を明記しており、ブラックボックス検証という本ラボと同じスタンス

**矛盾・相違ポイント**

- **一致条件を「部分一致」と報告している。** 本ラボは完全一致（`OP_TEST_TOKEN` 同士）のみを検証しており、部分一致の可否は未検証
- **識別手段を 3 つ（`name` / `id` / `label`）に限定している**のに対し、本ラボでは `placeholder` / `title` / `aria-label` / `aria-labelledby` / 近傍可視テキストでも充填されました。2021 年から 2026 年の間の挙動変化か、当時の調査範囲の差か、いずれかです
- 充填の**選定順序**（複数候補がある場合にどれが選ばれるか）には触れていない

### [Autofill using HTML "id"（1Password Community, 2023-01-12）](https://www.1password.community/1password-at-home-31/autofill-using-html-id-10427)

**裏付けポイント**

- 1Password 社員 steph_giles が HTML ID を使ったカスタムフィールドでの充填方法を案内 → **1-A（`id` 一致）**の裏付け
- 投稿者は最終的に動作したと報告しており、`id` 経路が実用上機能することが確認されている

**矛盾・相違ポイント**

- **「カスタムフィールドではインラインメニューが表示されない。ツールバーの 1Password アイコンから『Autofill』を選ぶ必要がある」**とスタッフが明言。本ラボもツールバーのポップアップから実行しており手順は一致する。ショートカットやインラインメニュー経由での差は未検証
- スタッフは HTML `name` 属性や `label` 要素の対応可否については明言を避けており、mizdra 氏の報告や本ラボの 1-B / 2 系の結果を公式に裏付けるものではありません

### [Autofill of Custom Field（1Password Community, 2025-05-22）](https://www.1password.community/discussions/1password/autofill-of-custom-field/156349)

**裏付けポイント**

- 投稿者は `txtName` と `txtNumber` という**2 つのカスタムフィールドが同時に充填された**（Windows 上）と報告 → 本ラボの「1 回の Autofill で 1 フィールドのみ」という観測が**「カスタムフィールド 1 つにつき 1 フィールド」という意味である**ことを裏付けます。ページ全体で 1 つしか埋まらないという意味ではありません

**矛盾・相違ポイント**

- **Windows では動作し iOS では動作しない**というプラットフォーム差が報告されています。本ラボは macOS のデスクトップブラウザでしか検証しておらず、**結果をモバイルへ一般化できません**
- 公式スタッフの回答は追加情報の要求のみで、スレッド内に技術的な結論はありません。裏付けの強度は低いソースです

### [1Password で自動保存されないフィールドを入力する（基素, Scrapbox）](https://scrapbox.io/motoso/1Password%E3%81%A7%E8%87%AA%E5%8B%95%E4%BF%9D%E5%AD%98%E3%81%95%E3%82%8C%E3%81%AA%E3%81%84%E3%83%95%E3%82%A3%E3%83%BC%E3%83%AB%E3%83%89%E3%82%92%E5%85%A5%E5%8A%9B%E3%81%99%E3%82%8B)

**裏付けポイント**

- 入力欄の `name` 属性に合わせたカスタムフィールドを作ることで充填できる、という手順 → **1-B（`name` 一致）**と整合

**矛盾・相違ポイント**

- 手順の記録が主目的で、網羅的な検証ではありません。他の識別手段や選定順序についての情報はなく、**本ラボの結果を否定も補強もしません**。裏付けの強度は低いソースです

### [How to turn off password managers for fields（Stefan Judis）](https://www.stefanjudis.com/snippets/turn-off-password-managers/)

**裏付けポイント**

- 1Password の無効化には `data-1p-ignore` が必要（LastPass は `data-lpignore`、Bitwarden は `data-bwignore`）と整理 → **`autocomplete=off` では充填を防げない**という 6-C の結果と整合

**矛盾・相違ポイント**

- **`data-1p-ignore` の効果自体は本ラボで未検証**です。この属性が実際に充填を止めるかは確認していません

## 検証できなかったソース

- **「Autocomplete ignores textarea fields」（1Password Community, discussion/138748）** — 検索結果には「textarea では動かず、text input に変えると動く」という報告として現れましたが、**URL がリンク切れ（404）で本文を確認できませんでした**。5-A（textarea 無効）の裏付けとしては強度が低く、検索結果のスニペット以上の確認が取れていません
- **「Why Password Managers Ignore Input Fields」（Medium, Thomas Gamauf）** — **HTTP 403 で取得できず**、内容を確認していません
- 検索結果に現れた `alibaba.com` の「product-insights」系ページは、出典不明の統計（「サポートチケットが 340% 増加」等）を含む生成コンテンツと判断し、**ソースとして採用していません**

## 突き合わせから追加したいテストケース

外部ソースの記述のうち、本ラボで未カバーの項目です。

1. **カスタムフィールドの種別違い**（コード変更不要） — メール / パスワード / 電話 / 住所 種別の `OP_TEST_TOKEN` を作り、既存の 5 系（`?only=5-A` / `5-D` / `5-E`）を再実行する。詳細は [findings.md](findings.md) を参照
2. **部分一致** — `name="prefix_OP_TEST_TOKEN_suffix"` のように、カスタムフィールド名を部分文字列として含むケース
3. **`data-1p-ignore` / `data-op-ignore`** — 一致する識別情報を持ちつつ、この属性で除外されるかを確認するケース
4. **隠し input 自体への充填** — `style="display:none"` の input に、一致する識別情報を与えたケース

## その他の参考リンク

- [Customize your 1Password items（1Password Support）](https://support.1password.com/custom-fields/) — カスタムフィールドの基本仕様
- [Change where a login is suggested and filled（1Password Support）](https://support.1password.com/autofill-behavior/) — URL マッチング（eTLD+1、Public Suffix List）の仕様
