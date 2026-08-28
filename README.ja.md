# DESIGN.md Sync — Figma ⇄ design.md ブリッジ（Figma Design Agent 用）

Figma のカスタムスキル登録機能に登録して、チャットから実行するスキルです。[Google の design.md仕様](https://github.com/google-labs-code/design.md/blob/main/docs/spec.md)に準拠したDESIGN.mdファイルと、Figmaファイルのデザイントークンを双方向に同期します — ImportはDESIGN.mdから変数・テキストスタイル・コンポーネントを構築し、ExportはFigmaファイルの既存トークンを読み取ってDESIGN.mdを生成します。

> *English README: see [README.md](README.md).*

> **有料講座について**: このスキルは有料講座「[KMRVID Figma Skills](https://kmrvid-claude-skills.gaspanik.workers.dev/figma/)」に収録されている21スキルの1つと同一内容です（機能を削った軽量版ではありません）。全スキルカタログはこちら: [KMRVID Figma Skills README](https://fragrant-edam-563.notion.site/KMRVID-Figma-Skills-README-md-3ad5dae25dd280d69c12c69b277d90b6)

> **有料講座のプレビュー**: 講座内の各種スキルを実際に動かしている様子は、こちらの[YouTube](https://www.youtube.com/@kmrvid/videos)や講座の[無料プレビュー](https://kmrvid.com/apps/ldt-course/course/6922c0ad336ff5545bbff61d/6a6b17c49dc4e0183716f605?locale=ja)（アカウント作成、ログイン不要）に掲載しております。

---

## これは何か

design.md仕様は、デザインシステムを1つのMarkdownファイルとして表現します — トークン（カラー・タイポグラフィ・スペーシング・角丸・コンポーネント）はYAML frontmatterに、その意図はプロース（本文）に書かれます。このスキルは、そのファイルと実際のFigmaファイルをつなぐ橋渡し役です。

- **Import** — DESIGN.md（自分で書いたもの、チームメンバーのもの、サードパーティのテンプレートなど）を添付すると、対応する変数コレクション・テキストスタイル・コンポーネントをFigma上に構築し、そのトークンをすぐにデザイン作業で使える状態にします
- **Export** — すでにFigma上に変数・テキストスタイルがある場合は、それらを読み取ってDESIGN.mdを生成します。定義されているのに一度もコンポーネントから参照されていない色がないかをチェックする整合性チェックも含みます

```
DESIGN.mdが添付されているか？
        │
   ┌────┴────┐
  はい          いいえ → Import/Exportどちらか質問
   │
   ▼
Import: トークン解析（frontmatterまたはプロースから推測）→ 変数
コレクション作成 → テキストスタイル作成（フォント代替対応）→
コンポーネント作成 → プレビューページ（任意）→ 件数付き完了報告

Export: 変数/テキストスタイル/コンポーネントを収集 → YAML
frontmatter＋本文を生成 → 整合性チェック（全色が参照済みか／
本文がトークンと一致しているか）→ プレビューページ（任意）→ 出力
```

作成・バインドを行う各ステップは、APIコールが成功したかどうかだけでなく、実際に作成・バインドされた値を再取得して確認してから件数を報告します。

---

## スコープ

- **ベーストークンとコンポーネントの基本状態のみが対象です。** 完全なデザインシステムの状態管理ツールではありません — カラー・タイポグラフィ・スペーシング・角丸、そして各コンポーネントのデフォルト外観を同期します。hover/active/focus/disabledなどのインタラクション状態やバリアントの網羅は対象外です。その部分はStorybookなど専用のツールと併用してください
- **厳密なfrontmatter形式でないDESIGN.mdにも対応します。** 実際に流通しているDESIGN.mdが必ずしもYAML frontmatter形式に従っているとは限りません — frontmatterがない見出しベースのプロースMarkdownからも、同じトークンデータを推測して読み取ります
- **モデルに合わないコンポーネントカテゴリはスキップされることがあります。** 例えば、角丸0pxのシステムの中に存在する円形要素など。Importはこれを無理に押し込まず、完了報告で理由とともに明示します
- **`colors.primary`は仕様上必須です。** 元ファイルに`primary`が定義されていない場合、Exportはファイル内で実質的にその役割を担っている色を代わりに推測して補完し、報告に明記します

---

## 必要要件

- **Figma（Design Agent / カスタムスキル登録機能）** — このスキルはFigma内蔵のPlugin APIスクリプト実行ツール（`evaluate_script`等）のみを使用します
- 外部のMCPサーバーやAPIキーは不要です。SKILL.md単体をFigma Design Agentのカスタムスキルとして登録するだけで動作します

---

## リポジトリ構成

```
skills/
  design-md/
    SKILL.md          — Figmaのカスタムスキル登録機能に登録するスキル定義
    LICENSE
```

---

## はじめかた

**Figma AI Skillsコミュニティへの公開後は**、[AI Skillsライブラリ](https://www.figma.com/community/ai-skills)からダウンロード不要で直接追加できるようになります。それまではソースから登録してください。

**ソースから:** このリポジトリをcloneしてスキルファイルを登録します。

```bash
git clone https://github.com/gaspanik/design-md-skill
```

**1. Figmaのカスタムスキル登録機能で `skills/design-md/SKILL.md` を登録**

**2. チャットからスキルを実行**

```
/design-md
```

```
このDESIGN.mdをImportして変数とコンポーネントを作って
```

```
Export the current file's variables and components as a DESIGN.md
```

DESIGN.mdファイルを添付すればそのままImportモードへ、何も添付せず実行すればImport/Exportどちらを行うか質問されます。

---

## 実行時の流れ

1. **モード判定** — チャットにDESIGN.mdが添付されていればそのままImport、なければImport/Exportどちらか質問
2. **Import** — frontmatterからトークンを解析（なければプロースから推測）→ 変数コレクション作成 → フォント代替対応込みでテキストスタイル作成 → 変数バインド済みでコンポーネント作成 → プレビューページ（任意）
3. **Export** — ローカルの変数・テキストスタイル・コンポーネントを全て収集 → YAML frontmatter＋本文を生成 → 整合性チェック（全色がコンポーネントから参照されているか／本文がトークンと一致しているか）を実行
4. **検証** — 各書き込みステップは、作成した内容を再取得して実際にバインドされた値を確認してから件数を報告します。APIコールの成功だけをもって完了とはしません
5. **完了報告** — トークン・コンポーネントの件数、スキップされたセクションやコンポーネントカテゴリとその理由、フォント代替の有無、プレビューページ作成の有無を報告

---

## 出力例

```
Import 完了報告 — RawBlock Design System

検出セクション: Colors ✓ / Typography ✓ / Spacing ✓ / Border Radius ✓ / Components ✓

変数コレクション: 4コレクション作成、変数34個 全数一致
テキストスタイル: 8個作成（fontSizeバインド 8/8確認、フォント代替なし）
コンポーネント: 11個作成（カラーfillsバインド 8/10、Ghost・Inputは意図的にtransparent）
```

---

Built by Masaaki Komori - [@cipher](https://x.com/cipher) · Skill for [Figma](https://www.figma.com/)
