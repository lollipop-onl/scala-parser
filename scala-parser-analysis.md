# Prettier Scala Parser 実装分析レポート

## 目次
1. [概要](#概要)
2. [Prettier Javaパーサーのアーキテクチャ分析](#prettier-javaパーサーのアーキテクチャ分析)
3. [ScalaとJavaの構文比較](#scalaとjavaの構文比較)
4. [Scalaパーサー実装要件](#scalaパーサー実装要件)
5. [実装推奨事項](#実装推奨事項)

## 概要

このドキュメントは、Prettier JavaパーサーをベースにScalaパーサーを実装するための分析結果をまとめたものです。Prettier Javaの実装を詳細に調査し、Scala固有の構文と機能をサポートするために必要な拡張点を特定しました。

## Prettier Javaパーサーのアーキテクチャ分析

### 1. プロジェクト構造

Prettier Javaは、monorepoとして構成されており、主に2つのパッケージから成り立っています：

```
prettier-java/
├── packages/
│   ├── java-parser/          # パーサーのコア実装
│   │   ├── src/
│   │   │   ├── index.js      # エントリーポイント
│   │   │   ├── lexer.js      # レクサー実装
│   │   │   ├── parser.js     # パーサー実装
│   │   │   ├── tokens.js     # トークン定義
│   │   │   └── productions/  # 文法ルール定義
│   │   │       ├── arrays.js
│   │   │       ├── blocks-and-statements.js
│   │   │       ├── classes.js
│   │   │       ├── expressions.js
│   │   │       ├── interfaces.js
│   │   │       ├── lexical-structure.js
│   │   │       ├── names.js
│   │   │       ├── packages-and-modules.js
│   │   │       └── types-values-and-variables.js
│   │   └── package.json
│   └── prettier-plugin-java/  # Prettierプラグイン実装
│       ├── src/
│       │   ├── index.ts      # プラグインエントリーポイント
│       │   ├── parser.ts     # パーサー統合
│       │   ├── printer.ts    # プリンターメイン
│       │   └── printers/     # プリンター実装
│       └── package.json
└── lerna.json
```

### 2. 主要技術と実装パターン

#### Chevrotainパーサーライブラリ
- **LL(\*)パーサー**: LLStarLookaheadStrategyを使用
- **CST（具象構文木）ベース**: コメントや空白の保持が容易
- **バックトラッキング**: 複雑な文法の曖昧性解決に使用

#### レクサー実装
```javascript
// tokens.jsでのトークン定義例
const Identifier = createToken({
  name: "Identifier",
  pattern: matchJavaIdentifier,  // カスタムマッチャー関数
  categories: [identifierTokens.Identifier]
});

// lexer.jsでのレクサー作成
const JavaLexer = new Lexer(allTokens, {
  ensureOptimizations: true,
  skipValidations: true
});
```

#### パーサー実装
```javascript
class JavaParser extends CstParser {
  constructor() {
    super(allTokens, {
      nodeLocationTracking: "full",
      lookaheadStrategy: new LLStarLookaheadStrategy()
    });
    // 各文法カテゴリーのルールを定義
    defineRules.call(this);
  }
}
```

#### プリンター実装
```typescript
// Prettierのビルダーを使用したフォーマット
function printClassDeclaration(path: AstPath, print: PrintFn) {
  return group([
    printModifiers(path, print),
    "class ",
    path.call(print, "identifier"),
    printTypeParameters(path, print),
    printExtends(path, print),
    printImplements(path, print),
    " ",
    path.call(print, "body")
  ]);
}
```

### 3. 実装の特徴

- **モジュール化**: 文法カテゴリーごとにファイルを分割
- **言語仕様準拠**: Java Language Specification (JLS) への明確な参照
- **パフォーマンス最適化**: skipValidationsとensureOptimizations
- **完全な位置情報**: nodeLocationTracking: "full"
- **コメント処理**: 特殊なコメント処理とprettier-ignoreのサポート

## ScalaとJavaの構文比較

### 1. 基本構文の違い

| 機能 | Java | Scala |
|------|------|-------|
| 変数宣言 | `int x = 10;`<br>`final int y = 20;` | `var x = 10`<br>`val y = 20` |
| 関数定義 | `public int add(int a, int b) {`<br>`  return a + b;`<br>`}` | `def add(a: Int, b: Int): Int = a + b` |
| クラス | `class Person {`<br>`  private String name;`<br>`  public Person(String name) {`<br>`    this.name = name;`<br>`  }`<br>`}` | `class Person(val name: String)` |
| インポート | `import java.util.List;` | `import java.util.{List, Map}`<br>`import java.util._` |

### 2. Scala固有の構文要素

#### パターンマッチング
```scala
x match {
  case 1 => "one"
  case n if n > 0 => "positive"
  case _ => "other"
}
```

#### ケースクラス
```scala
case class Point(x: Int, y: Int)
// 自動的にequals, hashCode, toString, copy, apply/unapplyが生成される
```

#### トレイトとミックスイン
```scala
trait Printable {
  def print(): Unit
}

class Document extends Printable with Debuggable {
  def print() = println("document")
}
```

#### 暗黙の変換（implicit）
```scala
implicit val defaultTimeout: Int = 5000
def request(url: String)(implicit timeout: Int) = ???

implicit class RichString(s: String) {
  def exclaim = s + "!"
}
```

#### for内包表記
```scala
val pairs = for {
  x <- 1 to 5
  y <- 1 to 5
  if x != y
} yield (x, y)
```

#### 文字列補間
```scala
val name = "Alice"
s"Hello, $name!"
f"Pi is approximately $math.Pi%.2f"
raw"Line1\nStill Line1"
```

### 3. パーサー実装時の注意点

1. **セミコロン推論**: Scalaは改行で文が終わると推論する
2. **演算子の多様性**: 任意の記号の組み合わせが演算子になりうる
3. **文脈依存の構文**: `_`の意味が文脈により変化
4. **中置記法**: `1 to 10`は`1.to(10)`と同じ
5. **すべてが式**: if、match、tryなどすべてが値を返す

## Scalaパーサー実装要件

### 1. トークン定義の拡張

#### Scala固有のキーワード
```javascript
const scalaKeywords = [
  // 基本キーワード
  "val", "var", "def", "object", "trait", "implicit", "lazy",
  "sealed", "override", "case", "match", "with", "yield",
  
  // 型関連
  "type", "forSome", "given", "using", "extension", "derives",
  "opaque", "transparent", "inline", "infix",
  
  // その他
  "abstract", "final", "private", "protected", "super", "this"
];
```

#### Scala特有の演算子
```javascript
const scalaOperators = {
  "<-":    "LeftArrow",        // for内包表記
  "=>":    "Arrow",            // 関数リテラル、case節
  "::":    "ColonColon",       // List cons
  ":::":   "TripleColon",      // List concatenation
  "->":    "RightArrow",       // タプル生成
  "<:":    "UpperBound",       // 上限境界
  ">:":    "LowerBound",       // 下限境界
  "<%":    "ViewBound",        // ビュー境界（Scala 2.x）
  "#":     "TypeProjection",   // 型投影
  "@":     "At",               // アノテーション
  "_":     "Underscore"        // プレースホルダー
};
```

#### 文字列補間のトークン
```javascript
// モード切り替えが必要
const StringInterpolationMode = "StringInterpolation";
const TemplateMode = "Template";

createToken({ 
  name: "StringInterpolationBegin", 
  pattern: /[sf]"/, 
  push_mode: StringInterpolationMode 
});
```

### 2. パーサールールの追加

#### パターンマッチング
```javascript
$.RULE("matchExpression", () => {
  $.SUBRULE($.expression);
  $.CONSUME(t.Match);
  $.CONSUME(t.LCurly);
  $.MANY(() => {
    $.SUBRULE($.caseClause);
  });
  $.CONSUME(t.RCurly);
});

$.RULE("pattern", () => {
  $.OR([
    { ALT: () => $.SUBRULE($.literalPattern) },
    { ALT: () => $.SUBRULE($.variablePattern) },
    { ALT: () => $.SUBRULE($.typedPattern) },
    { ALT: () => $.SUBRULE($.constructorPattern) },
    { ALT: () => $.SUBRULE($.tuplePattern) },
    { ALT: () => $.SUBRULE($.wildcardPattern) }
  ]);
});
```

#### トレイト定義
```javascript
$.RULE("traitDeclaration", () => {
  $.MANY(() => $.SUBRULE($.modifier));
  $.CONSUME(t.Trait);
  $.SUBRULE($.identifier);
  $.OPTION(() => $.SUBRULE($.typeParameters));
  $.OPTION2(() => $.SUBRULE($.traitTemplateOpt));
});

$.RULE("withClause", () => {
  $.CONSUME(t.With);
  $.SUBRULE($.annotType);
  $.MANY(() => {
    $.CONSUME2(t.With);
    $.SUBRULE2($.annotType);
  });
});
```

#### for内包表記
```javascript
$.RULE("forExpression", () => {
  $.CONSUME(t.For);
  $.OR([
    { ALT: () => $.CONSUME(t.LBrace) },
    { ALT: () => $.CONSUME(t.LCurly) }
  ]);
  $.SUBRULE($.enumerators);
  $.OR2([
    { ALT: () => $.CONSUME2(t.RBrace) },
    { ALT: () => $.CONSUME2(t.RCurly) }
  ]);
  $.OPTION(() => {
    $.CONSUME(t.Yield);
    $.SUBRULE($.expression);
  });
});
```

### 3. 特殊処理の実装

#### セミコロン推論
```javascript
// トークン後処理でセミコロンを挿入
function insertSemicolons(tokens) {
  const result = [];
  for (let i = 0; i < tokens.length; i++) {
    result.push(tokens[i]);
    
    if (shouldInsertSemicolon(tokens, i)) {
      result.push({
        image: ";",
        tokenType: VirtualSemicolon,
        startOffset: tokens[i].endOffset,
        endOffset: tokens[i].endOffset
      });
    }
  }
  return result;
}
```

#### 中置記法のサポート
```javascript
$.RULE("infixExpression", () => {
  $.SUBRULE($.prefixExpression);
  $.MANY(() => {
    // 演算子の優先順位を考慮
    $.SUBRULE($.infixOp);
    $.SUBRULE2($.prefixExpression);
  });
});

// 右結合演算子の特別処理
$.RULE("consExpression", () => {
  $.SUBRULE($.simpleExpression);
  $.OPTION(() => {
    $.CONSUME(t.ColonColon);
    $.SUBRULE($.consExpression); // 右再帰
  });
});
```

### 4. プリンター実装の考慮事項

#### Scalaスタイルガイド準拠
```typescript
const scalaFormatOptions = {
  // インデント
  indentWidth: 2,
  
  // 中括弧の配置
  bracePlacement: {
    classDeclaration: "same-line",
    methodDeclaration: "same-line",
    matchExpression: "same-line"
  },
  
  // 空白の挿入
  spaceBeforeColon: false,
  spaceAfterColon: true,
  spaceAroundOperators: true,
  
  // 改行
  maxLineLength: 100,
  alignMultilineParameters: true
};
```

#### ケース固有のフォーマット
```typescript
// パターンマッチのフォーマット
function printMatchExpression(path: AstPath, print: PrintFn) {
  return group([
    path.call(print, "expression"),
    line,
    "match ",
    "{",
    indent([
      hardline,
      join(hardline, path.map(print, "cases"))
    ]),
    hardline,
    "}"
  ]);
}

// for内包表記のフォーマット
function printForExpression(path: AstPath, print: PrintFn) {
  const hasMultipleEnumerators = path.node.enumerators.length > 1;
  const delimiter = hasMultipleEnumerators ? hardline : line;
  
  return group([
    "for ",
    hasMultipleEnumerators ? "{" : "(",
    indent([
      delimiter,
      join([";", line], path.map(print, "enumerators"))
    ]),
    delimiter,
    hasMultipleEnumerators ? "}" : ")",
    path.node.hasYield ? " yield" : "",
    line,
    path.call(print, "body")
  ]);
}
```

## 実装推奨事項

### 1. アーキテクチャ設計

1. **モジュール構造**
   - java-parserと同様のmonorepo構造を採用
   - scala-parserとprettier-plugin-scalaの2パッケージ構成
   - 文法カテゴリーごとのファイル分割

2. **技術選定**
   - パーサーライブラリ: Chevrotain（Javaと同じ）
   - 言語: TypeScript（型安全性のため）
   - テストフレームワーク: Jest + Scala公式テストスイート

### 2. 実装の優先順位

#### フェーズ1: 基本構文（MVP）
- 基本的な式と文
- クラスとオブジェクト定義
- 関数定義
- 変数宣言
- 基本的な型システム

#### フェーズ2: Scala特有機能
- パターンマッチング
- for内包表記
- トレイトとミックスイン
- ケースクラス
- 文字列補間

#### フェーズ3: 高度な機能
- 暗黙の変換
- 型パラメータと境界
- マクロ（基本サポート）
- XMLリテラル（Scala 2.x）
- 高カインド型

### 3. 開発のベストプラクティス

1. **テスト駆動開発**
   - Scala公式のパーサーテストスイートを参考に
   - 各構文要素に対する包括的なテスト
   - エッジケースとエラーケースのテスト

2. **段階的な実装**
   - 最小限の動作するパーサーから開始
   - 機能を徐々に追加
   - 各段階でのリグレッションテスト

3. **パフォーマンス考慮**
   - Chevrotainの最適化機能を活用
   - 大規模なScalaプロジェクトでのベンチマーク
   - メモリ使用量の監視

4. **コミュニティとの連携**
   - Scala公式スタイルガイドとの整合性
   - 既存のScalaフォーマッターとの比較
   - ユーザーフィードバックの収集

### 4. 既知の課題と対策

1. **セミコロン推論の複雑さ**
   - 詳細なルールセットの実装
   - エッジケースの特定とテスト
   - ユーザー設定による調整可能性

2. **演算子の優先順位**
   - Scala仕様に準拠した優先順位テーブル
   - カスタム演算子への対応
   - 中置記法の適切な処理

3. **文脈依存の構文**
   - パーサーステートの管理
   - シンボルテーブルの構築
   - 型推論との連携（限定的）

## まとめ

Prettier Javaパーサーは、Scalaパーサー実装の優れた基盤となります。Chevrotainライブラリの採用、モジュール化されたアーキテクチャ、CST（具象構文木）ベースのアプローチは、Scalaの複雑な構文をサポートするのに適しています。

主な拡張点は：
1. Scala固有のトークンとキーワードの追加
2. パターンマッチング、for内包表記などの新しい構文ルール
3. セミコロン推論と中置記法のサポート
4. Scalaスタイルガイドに準拠したフォーマッター

段階的な実装アプローチと、包括的なテストスイートの構築により、高品質なScalaフォーマッターの開発が可能です。