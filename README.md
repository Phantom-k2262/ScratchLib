# ScratchLib

Scratch で不足している低レベル機能や、複数スプライトから再利用したい共通処理を **ライブラリとして切り出して管理するためのリポジトリ**です。

Scratch のカスタムブロックは基本的にスプライト内へ閉じており、別スプライトから直接関数として呼び出すことができません。そのため本リポジトリでは、必要に応じて **Broadcast を公開インターフェースとして利用し、共有 `args` リストと `g_return` を呼出規約（ABI）として扱う**設計を採用します。

## Libraries

| Library | 概要 | Status |
|---|---|---|
| `Stdmem` | Scratch の List を仮想ヒープとして利用するメモリ管理ライブラリ。Best-Fit `malloc` / `free`、ポインタ風 `read` / `write`、イベント RPC インターフェースを提供 | Experimental |

詳細設計: [Stdmem Design](docs/stdmem-design.md)

## Design policy

### 1. ライブラリはスプライト単位で分離する

ライブラリ固有の状態と内部処理は、原則としてライブラリ用スプライトへ閉じ込めます。

```text
Stdmem
├─ allocator core
├─ RPC adapter
├─ private heap state
└─ diagnostics
```

アプリケーション側のスプライトは、ライブラリ内部の変数や実装詳細へ直接依存しないことを目標とします。

### 2. 外部公開 API はイベントを利用する

別スプライトからカスタムブロックを直接呼べないため、公開 API は Broadcast イベントとして表現します。

イベント命名規則:

```text
<UpperCamelCase Sprite>.<camelCase function>
```

例:

```text
Stdmem.init
Stdmem.malloc
Stdmem.free
Stdmem.read
Stdmem.write
Stdmem.dump
```

### 3. 引数は `args`、戻り値は `g_return`

Broadcast 自体には引数がないため、可変長引数領域としてプロジェクトグローバルの `args` List を利用します。

```text
args[1] = arg0
args[2] = arg1
args[3] = arg2
...
```

戻り値が必要な API は `g_return` へ結果を書き込みます。

呼び出し例:

```text
args をすべて削除
args に 16 を追加

Stdmem.malloc を送って待つ

ptr = g_return
```

`args` と `g_return` は共有領域なので、現在の ABI は **single-flight（同時に1呼び出し）** を前提とします。

### 4. 公開イベントとコアロジックを分離する

イベント受信部は実装本体ではなく Interface / Adapter として扱います。

```text
Broadcast
  ↓
引数個数チェック
  ↓
args のスナップショット
  ↓
Parse / Cast
  ↓
Validation
  ↓
内部カスタムブロック
  ↓
Core Logic
```

この境界により、Scratch 特有のイベント制約をコアロジックへ持ち込まないことを狙います。

### 5. コメント用 no-op ブロック

Scratch 上でもコード中に意図を残しやすくするため、処理を行わないコメント用カスタムブロックを利用します。

```text
/* (comment) */
// (comment)
```

- `/* ... */`: 関数・セクション単位の説明
- `// ...`: 直後の処理、制約、不変条件の説明

## Repository layout

ライブラリが増えた場合は、概ね次の形で管理する想定です。

```text
ScratchLib/
├─ README.md
├─ docs/
│  ├─ stdmem-design.md
│  └─ ...
├─ stdmem/
│  ├─ Stdmem.sprite3
│  ├─ examples/
│  └─ ...
└─ ...
```

## Scope

このリポジトリは Scratch を別言語へ置き換えることを目的としていません。

Scratch の制約を前提にしつつ、以下を実験・整理することを目的とします。

- ライブラリ化
- 疎結合化
- スコープ管理
- 明示的な API / ABI
- データ構造
- メモリモデル
- エンジン設計
- テスタビリティ
- 他言語で一般的な設計概念の Scratch への適用

## License

ライセンスを設定する場合は、このセクションを更新してください。
