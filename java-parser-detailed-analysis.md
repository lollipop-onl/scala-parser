# Java Parser 詳細実装分析

## 目次
1. [概要](#概要)
2. [トークン定義](#トークン定義)
3. [構文ルール実装](#構文ルール実装)
4. [Javaの構文要素と実装の対応表](#javaの構文要素と実装の対応表)
5. [実装の特徴と技術](#実装の特徴と技術)
6. [JavaとScalaの構文対応](#javaとscalaの構文対応)

## 概要

このドキュメントは、prettier-javaパーサーの実装を詳細に分析し、Javaのすべての構文要素がどのように実装されているかを記録したものです。Scalaパーサー実装の参考資料として使用します。

## トークン定義

### 1. キーワードトークン

#### 通常のキーワード（53個）
```javascript
// 制御フロー
break, continue, do, else, for, if, return, switch, while

// 例外処理
assert, catch, finally, throw, throws, try

// クラス・インターフェース関連
abstract, class, extends, implements, interface, new, super, this

// アクセス修飾子
private, protected, public

// その他の修飾子
final, native, static, strictfp, synchronized, transient, volatile

// 型
boolean, byte, char, double, float, int, long, short, void

// 特殊
case, const, default, enum, goto, import, instanceof, package

// アンダースコア（Java 9から予約語）
_
```

#### 制限付きキーワード（コンテキスト依存）
```javascript
// モジュール関連（Java 9+）
module, open, requires, transitive, exports, opens, to, uses, provides, with

// Sealed classes（Java 17+）
sealed, non-sealed, permits

// その他
when
```

#### 特殊な識別子
```javascript
var    // ローカル変数型推論（Java 10+）
yield  // switch式（Java 14+）
record // レコード型（Java 14+）
```

#### リテラル
```javascript
true, false, null
```

### 2. 演算子トークン

#### 代入演算子
| 演算子 | 説明 |
|--------|------|
| `=` | 単純代入 |
| `+=` | 加算代入 |
| `-=` | 減算代入 |
| `*=` | 乗算代入 |
| `/=` | 除算代入 |
| `%=` | 剰余代入 |
| `&=` | ビットAND代入 |
| `\|=` | ビットOR代入 |
| `^=` | ビットXOR代入 |
| `<<=` | 左シフト代入 |
| `>>=` | 符号付き右シフト代入 |
| `>>>=` | 符号なし右シフト代入 |

#### 二項演算子
| カテゴリ | 演算子 |
|----------|--------|
| 算術 | `+`, `-`, `*`, `/`, `%` |
| 比較 | `<`, `>`, `<=`, `>=`, `==`, `!=` |
| 論理 | `&&`, `\|\|` |
| ビット | `&`, `\|`, `^` |
| シフト | `<<`, `>>`, `>>>` |
| 型チェック | `instanceof` |

#### 単項演算子
| 種類 | 演算子 |
|------|--------|
| 前置 | `++`, `--`, `+`, `-`, `!`, `~` |
| 後置 | `++`, `--` |

### 3. 区切り記号

| 記号 | 用途 |
|------|------|
| `@` | アノテーション |
| `.` | メンバアクセス |
| `...` | 可変長引数 |
| `,` | 要素の区切り |
| `;` | 文の終端 |
| `::` | メソッド参照 |
| `()` | 括弧（グループ化、パラメータ） |
| `{}` | ブロック |
| `[]` | 配列インデックス |
| `->` | ラムダ式 |
| `:` | 三項演算子、ラベル |
| `?` | 三項演算子、ワイルドカード |

### 4. リテラルトークン

#### 数値リテラル
| 種類 | パターン例 | 説明 |
|------|------------|------|
| 2進数 | `0b1010`, `0B1111_0000L` | 2進数表記 |
| 8進数 | `0777`, `0_123L` | 8進数表記 |
| 10進数 | `123`, `1_000_000L` | 10進数表記 |
| 16進数 | `0xFF`, `0x1234_5678L` | 16進数表記 |
| 浮動小数点 | `3.14`, `1.0e-10f` | 浮動小数点 |
| 16進浮動小数点 | `0x1.8p1` | 16進浮動小数点 |

#### 文字列リテラル
| 種類 | 例 | 説明 |
|------|-----|------|
| 文字 | `'a'`, `'\n'`, `'\u0041'` | 単一文字 |
| 文字列 | `"Hello"`, `"Line\nBreak"` | 通常の文字列 |
| テキストブロック | `"""多行文字列"""` | Java 15+ |
| 文字列テンプレート | `STR."Hello \{name}"` | Java 21+ (Preview) |

## 構文ルール実装

### 1. lexical-structure.js - 字句構造

| ルール名 | Java構文 | 説明 |
|----------|----------|------|
| `literal` | リテラル全般 | すべてのリテラル型の選択 |
| `integerLiteral` | `123`, `0xFF` | 整数リテラル（2進、8進、10進、16進） |
| `floatingPointLiteral` | `3.14f` | 浮動小数点リテラル |
| `booleanLiteral` | `true`, `false` | 真偽値リテラル |
| `shiftOperator` | `<<`, `>>`, `>>>` | シフト演算子（特殊処理付き） |

### 2. types-values-and-variables.js - 型、値、変数

| ルール名 | Java構文 | 説明 |
|----------|----------|------|
| `primitiveType` | `int`, `boolean` | プリミティブ型 |
| `numericType` | `int`, `double` | 数値型 |
| `referenceType` | `String`, `List<T>` | 参照型 |
| `classOrInterfaceType` | `HashMap<K,V>` | クラス/インターフェース型 |
| `typeVariable` | `T`, `E` | 型変数 |
| `dims` | `[]`, `[][]` | 配列の次元 |
| `typeParameter` | `<T extends Comparable>` | 型パラメータ定義 |
| `typeArguments` | `<String, Integer>` | 型引数 |
| `wildcard` | `?`, `? extends T` | ワイルドカード型 |

### 3. names.js - 名前

| ルール名 | Java構文 | 説明 |
|----------|----------|------|
| `moduleName` | `com.example.module` | モジュール名 |
| `packageName` | `java.util` | パッケージ名 |
| `typeName` | `String`, `Map.Entry` | 型名 |
| `expressionName` | `obj.field` | 式名 |
| `methodName` | `toString` | メソッド名 |
| `packageOrTypeName` | `java.util.List` | パッケージまたは型名 |
| `ambiguousName` | 文脈依存の名前 | 曖昧な名前 |

### 4. packages-and-modules.js - パッケージとモジュール

| ルール名 | Java構文 | 説明 |
|----------|----------|------|
| `compilationUnit` | Javaファイル全体 | コンパイル単位 |
| `packageDeclaration` | `package com.example;` | パッケージ宣言 |
| `importDeclaration` | `import java.util.*;` | インポート宣言 |
| `typeDeclaration` | クラス/インターフェース宣言 | 型宣言 |
| `moduleDeclaration` | `module mymodule { }` | モジュール宣言 |
| `requiresModuleDirective` | `requires java.base;` | requires指令 |
| `exportsModuleDirective` | `exports com.example;` | exports指令 |
| `opensModuleDirective` | `opens com.example;` | opens指令 |
| `usesModuleDirective` | `uses MyService;` | uses指令 |
| `providesModuleDirective` | `provides MyService with MyImpl;` | provides指令 |

### 5. classes.js - クラス

| ルール名 | Java構文 | 説明 |
|----------|----------|------|
| `classDeclaration` | `class MyClass { }` | クラス宣言 |
| `normalClassDeclaration` | `public class A extends B { }` | 通常のクラス宣言 |
| `classModifier` | `public`, `abstract`, `final` | クラス修飾子 |
| `typeParameters` | `<T, U extends Number>` | 型パラメータ |
| `classExtends` | `extends BaseClass` | extends節 |
| `classImplements` | `implements Interface1, Interface2` | implements節 |
| `classPermits` | `permits SubClass1, SubClass2` | permits節（sealed classes） |
| `classBody` | `{ fields and methods }` | クラス本体 |
| `fieldDeclaration` | `private int count;` | フィールド宣言 |
| `methodDeclaration` | `public void method() { }` | メソッド宣言 |
| `constructorDeclaration` | `public MyClass() { }` | コンストラクタ宣言 |
| `enumDeclaration` | `enum Color { RED, GREEN }` | 列挙型宣言 |
| `recordDeclaration` | `record Point(int x, int y) { }` | レコード宣言 |

### 6. interfaces.js - インターフェース

| ルール名 | Java構文 | 説明 |
|----------|----------|------|
| `interfaceDeclaration` | `interface MyInterface { }` | インターフェース宣言 |
| `interfaceModifier` | `public`, `sealed` | インターフェース修飾子 |
| `interfaceExtends` | `extends Interface1, Interface2` | extends節 |
| `interfacePermits` | `permits Impl1, Impl2` | permits節 |
| `interfaceBody` | `{ methods and constants }` | インターフェース本体 |
| `constantDeclaration` | `int MAX_SIZE = 100;` | 定数宣言 |
| `interfaceMethodDeclaration` | `void method();` | インターフェースメソッド |
| `annotationInterfaceDeclaration` | `@interface MyAnnotation { }` | アノテーション型宣言 |
| `annotation` | `@Override`, `@Deprecated("msg")` | アノテーション使用 |

### 7. arrays.js - 配列

| ルール名 | Java構文 | 説明 |
|----------|----------|------|
| `arrayInitializer` | `{1, 2, 3}` | 配列初期化子 |
| `variableInitializerList` | `1, 2, 3` | 変数初期化子リスト |

### 8. blocks-and-statements.js - ブロックと文

| ルール名 | Java構文 | 説明 |
|----------|----------|------|
| `block` | `{ statements }` | ブロック |
| `blockStatement` | 文またはローカル宣言 | ブロック内の文 |
| `statement` | 各種文 | 文全般 |
| `ifStatement` | `if (cond) { } else { }` | if文 |
| `whileStatement` | `while (cond) { }` | while文 |
| `forStatement` | `for (;;) { }` | for文（通常/拡張） |
| `switchStatement` | `switch (val) { case 1: }` | switch文 |
| `tryStatement` | `try { } catch (E e) { }` | try文 |
| `pattern` | `String s`, `Point(int x, int y)` | パターン（Java 17+） |
| `guard` | `when (x > 0)` | パターンのガード条件 |

### 9. expressions.js - 式

| ルール名 | Java構文 | 説明 |
|----------|----------|------|
| `expression` | 式全般 | すべての式 |
| `lambdaExpression` | `x -> x * 2` | ラムダ式 |
| `conditionalExpression` | `a ? b : c` | 三項演算子 |
| `binaryExpression` | `a + b`, `a && b` | 二項演算式 |
| `unaryExpression` | `-x`, `!flag` | 単項演算式 |
| `primary` | リテラル、this、括弧式など | プライマリ式 |
| `methodInvocationSuffix` | `.method(args)` | メソッド呼び出し |
| `newExpression` | `new Class()` | インスタンス生成 |
| `castExpression` | `(Type) expr` | キャスト式 |
| `arrayCreationExpression` | `new int[10]` | 配列生成 |
| `template` | `STR."Hello \{name}"` | 文字列テンプレート（Java 21+） |

## 実装の特徴と技術

### 1. GATE関数の使用

GATEは条件付きでルールの一部を実行するための仕組みです。

```javascript
// 例：shiftOperator（lexical-structure.js）
$.CONSUME(t.Less);
$.OPTION({
  // 隣接する2つの'<'トークンのみ受け入れる
  GATE: () => tokenMatcher($.LA(1), t.Less) && 
              $.LA(0).endOffset + 1 === $.LA(1).startOffset,
  DEF: () => $.CONSUME2(t.Less)
});
```

### 2. BACKTRACK_LOOKAHEAD

より複雑な先読みが必要な場合に使用：

```javascript
// 例：dims（types-values-and-variables.js）
$.MANY({
  GATE: $.BACKTRACK_LOOKAHEAD($.isDims),
  DEF: () => {
    $.MANY2(() => $.SUBRULE($.annotation));
    $.CONSUME(t.LBracket);
    $.CONSUME(t.RBracket);
  }
});
```

### 3. 共通プレフィックスの抽出

パフォーマンス最適化のため、共通部分を先に処理：

```javascript
// 例：annotation（interfaces.js）
// 共通の'@'と型名を先に処理
const name = $.SUBRULE($.typeName);
// その後、3種類のアノテーションに分岐
```

### 4. tokenMatcherの活用

トークンの種類を効率的に比較：

```javascript
GATE: () => tokenMatcher($.LA(2), t.Star)
```

## JavaとScalaの構文対応

### 基本構文の対応

| Java構文 | Scala対応 | 実装の差異 |
|----------|-----------|------------|
| `class` | `class` | Scalaはプライマリコンストラクタを持つ |
| `interface` | `trait` | Scalaのtraitは実装を持てる |
| `extends` | `extends` | 同じだが、Scalaは1つのクラスのみ継承 |
| `implements` | `with` | Scalaは複数のtraitをmixin可能 |
| `final` | `final`/`val` | Scalaは変数レベルで不変性を指定 |
| `static` | `object`内 | Scalaはコンパニオンオブジェクトを使用 |
| `void` | `Unit` | 戻り値なしの表現が異なる |
| `;` | 省略可能 | Scalaはセミコロン推論を持つ |

### Scala固有の構文（Javaに存在しない）

| Scala構文 | 説明 | 実装の必要性 |
|-----------|------|--------------|
| `object` | シングルトンオブジェクト | 新規ルール必要 |
| `trait` | トレイト | interfaceを拡張して実装 |
| `case class` | ケースクラス | classDeclarationを拡張 |
| `match`/`case` | パターンマッチング | 新規ルール必要 |
| `for`/`yield` | for内包表記 | forStatementを大幅拡張 |
| `implicit` | 暗黙の変換/パラメータ | 修飾子として追加 |
| `lazy val` | 遅延評価 | 新規修飾子 |
| `def` | メソッド定義 | methodDeclarationを置換 |
| `val`/`var` | 変数宣言 | fieldDeclarationを拡張 |
| `=>` | 関数リテラル | lambdaExpressionを拡張 |
| `<-` | for内包表記の生成子 | 新規演算子 |
| `_` | プレースホルダ | 文脈依存の特殊処理 |
| 中置記法 | `a method b` | 式パーサーの大幅変更 |
| 文字列補間 | `s"Hello $name"` | 新規トークンモード |
| XMLリテラル | `<tag>content</tag>` | 特殊なレクサーモード |

### Java構文のScalaでの扱い

| Java構文 | Scalaでの扱い | 注意点 |
|----------|---------------|--------|
| `switch` | `match`で代替 | より強力なパターンマッチング |
| 配列 `[]` | `Array[T]` | ジェネリクス構文で統一 |
| `for(;;)` | `while`で代替 | Scalaのforは内包表記 |
| `instanceof` | パターンマッチで代替 | `case _: Type =>` |
| 三項演算子 `?:` | `if-else`式 | Scalaではif-elseが式 |
| `enum` | `sealed trait` + `case object` | より柔軟な代数的データ型 |
| `record` | `case class` | Scalaは以前から同等機能 |

### 実装の優先順位

1. **必須実装**（Scalaの基本機能）
   - `val`/`var`宣言
   - `def`によるメソッド定義
   - `object`宣言
   - `trait`定義
   - パターンマッチング（`match`/`case`）
   - for内包表記

2. **重要実装**（Scalaらしさ）
   - 中置記法
   - 文字列補間
   - `implicit`関連
   - ケースクラス
   - 型推論対応

3. **オプション実装**（高度な機能）
   - XMLリテラル（Scala 2.x）
   - マクロ関連
   - 高カインド型
   - パス依存型

この分析により、prettier-javaパーサーの実装パターンを活用しながら、Scala固有の構文を追加実装する必要があることが明確になりました。

## プリンター実装の詳細

### 1. プリンターアーキテクチャ

prettier-javaのプリンター実装は、以下のファイルで構成されています：

| ファイル名 | 役割 | 主要な処理内容 |
|----------|------|--------------|
| **index.ts** | メインエントリポイント | ノードタイプに基づくプリンター関数のディスパッチ |
| **helpers.ts** | 共通ユーティリティ | モディファイア処理、リスト処理、コメント処理 |
| **arrays.ts** | 配列関連 | 配列初期化子のフォーマット |
| **blocks-and-statements.ts** | ブロックと文 | 制御構文のフォーマット |
| **classes.ts** | クラス定義 | クラス、メソッド、フィールドのフォーマット |
| **expressions.ts** | 式 | 演算子、メソッド呼び出し、ラムダ式のフォーマット |
| **interfaces.ts** | インターフェース | インターフェース、アノテーションのフォーマット |
| **lexical-structure.ts** | 字句構造 | リテラル、テキストブロックのフォーマット |
| **names.ts** | 名前関連 | パッケージ名、型名のフォーマット |
| **packages-and-modules.ts** | パッケージとモジュール | import文、module宣言のフォーマット |
| **types-values-and-variables.ts** | 型と変数 | 型パラメータ、型引数のフォーマット |

### 2. Prettierビルダー関数の使用パターン

#### 基本的なビルダー関数

```typescript
// group - 要素をグループ化し、改行を最適化
group(["public", " ", "class", " ", className])

// indent - インデントレベルを増やす
indent([hardline, ...statements])

// join - 要素を区切り文字で結合
join([",", line], parameters)

// line - ソフト改行（スペースまたは改行）
["(", indent([softline, args]), softline, ")"]

// hardline - 強制改行
["{", indent([hardline, body]), hardline, "}"]

// softline - より柔軟な改行（改行またはスペースなし）
group([modifier, softline, declaration])

// ifBreak - 改行時のみ表示される要素
[list, ifBreak(",")] // trailing comma

// conditionalGroup - 複数のフォーマットパターンを試す
conditionalGroup([
  // 1行に収まる場合
  [prefix, " ", suffix],
  // 改行が必要な場合
  [prefix, indent([line, suffix])]
])
```

#### 高度なパターン

```typescript
// fill - 要素を可能な限り1行に詰め込む
fill(join(line, elements))

// lineSuffix - 行末に配置される要素
[statement, lineSuffix(" // comment")]

// dedent - インデントを減らす
dedent(contents)
```

### 3. 主要な処理パターン

#### モディファイアの処理（helpers.ts）

```typescript
export function printWithModifiers<T extends CstNode>(
  path: AstPath<T>,
  print: JavaPrintFn,
  modifierChild: P,
  contents: Doc,
  noTypeAnnotations = false
) {
  const declarationAnnotations: Doc[] = [];
  const typeAnnotations: Doc[] = [];
  const modifiers: string[] = [];
  
  // モディファイアを分類
  path.each(childPath => {
    const child = childPath.node;
    if (child.annotation) {
      if (isTypeAnnotation(child.annotation[0])) {
        typeAnnotations.push(call(childPath, print, "annotation"));
      } else {
        declarationAnnotations.push(call(childPath, print, "annotation"));
      }
    } else {
      modifiers.push(child.image);
    }
  }, modifierChild);
  
  // 正しい順序でソート
  modifiers.sort((a, b) => 
    indexByModifier.get(a)! - indexByModifier.get(b)!
  );
  
  return join(hardline, [
    ...declarationAnnotations,
    join(" ", [...modifiers, ...typeAnnotations, contents])
  ]);
}
```

#### ブロックの処理（blocks-and-statements.ts）

```typescript
export function printBlock(
  path: AstPath<JavaNonTerminal>,
  contents: Doc[]
) {
  if (!contents.length) {
    // 空ブロックの処理
    const danglingComments = printDanglingComments(path);
    return danglingComments.length
      ? ["{", indent([hardline, ...danglingComments]), hardline, "}"]
      : "{}";
  }
  
  // 通常のブロック
  return group([
    "{",
    indent([hardline, ...join(hardline, contents)]),
    hardline,
    "}"
  ]);
}
```

#### メソッドチェーンの処理（expressions.ts）

```typescript
// メソッドチェーンのインデント制御
const shouldIndent = 
  (methodInvocations.length > 1 || 
   !isPrimaryExpression(primary)) &&
  !isBuilderPattern(node);

methodInvocations.forEach((invocation, index) => {
  const doc = shouldIndent
    ? indent([index === 0 ? softline : hardline, invocation])
    : invocation;
  primary.push(doc);
});
```

#### 二項演算子の配置（expressions.ts）

```typescript
function binary(
  operands: Doc[],
  operators: { image: string; doc: Doc }[],
  options: {
    hasNonAssignmentOperators: boolean;
    isRoot: boolean;
    operatorPosition?: "start" | "end";
  }
) {
  const shouldBreak = options.hasNonAssignmentOperators;
  const parts: Doc[] = [];
  
  operands.forEach((operand, i) => {
    if (i > 0) {
      const operator = operators[i - 1];
      if (options.operatorPosition === "start") {
        parts.push([hardline, operator.doc, " ", operand]);
      } else {
        parts.push([" ", operator.doc, line, operand]);
      }
    } else {
      parts.push(operand);
    }
  });
  
  return shouldBreak ? group(parts) : parts;
}
```

### 4. コメント処理

#### コメントの分類

```typescript
// コメントの種類を判定
function isLeadingComment(comment: Comment): boolean {
  return comment.leading === true;
}

function isTrailingComment(comment: Comment): boolean {
  return comment.trailing === true;
}

function isDanglingComment(comment: Comment): boolean {
  return !comment.leading && !comment.trailing;
}
```

#### コメントに基づく改行制御

```typescript
// 前の要素との間に空行が必要かチェック
function shouldAddEmptyLine(
  node: CstNode,
  previous: CstNode | null
): boolean {
  if (!previous) return false;
  
  const nodeStartLine = lineStartWithComments(node);
  const prevEndLine = lineEndWithComments(previous);
  
  return nodeStartLine > prevEndLine + 1;
}
```

### 5. 特殊な処理

#### テキストブロックのインデント処理

```typescript
// テキストブロックの共通インデントを検出して除去
if (TextBlock) {
  const [open, ...lines] = TextBlock[0].image.split("\n");
  const baseIndent = findBaseIndent(lines);
  
  const processedLines = lines.map(line => 
    line.slice(baseIndent)
  );
  
  const textBlock = join(hardline, [open, ...processedLines]);
  
  // 文脈に応じてインデントを追加
  return ancestor?.name === "variableInitializer"
    ? indent(textBlock)
    : textBlock;
}
```

#### Switch式とSwitch文の区別

```typescript
// blocks-and-statements.tsのswitch処理
const isSwitchExpression = 
  path.getParentNode()?.name === "switchExpression";

if (isSwitchExpression) {
  // Switch式の場合は異なるフォーマット
  return group([
    "switch",
    " ",
    "(",
    indent([softline, selector]),
    softline,
    ")",
    " ",
    switchBlock
  ]);
} else {
  // Switch文の場合
  return [
    "switch",
    " ",
    "(",
    selector,
    ")",
    " ",
    switchBlock
  ];
}
```

### 6. Scalaプリンター実装への示唆

#### 必要な拡張ポイント

1. **中置記法のサポート**
   ```typescript
   // Scalaの中置記法用の特別な処理
   function printInfixExpression(left: Doc, op: Doc, right: Doc) {
     return group([left, line, op, line, right]);
   }
   ```

2. **パターンマッチングのフォーマット**
   ```typescript
   // match式の整形
   function printMatchExpression(expr: Doc, cases: Doc[]) {
     return group([
       expr,
       line,
       "match",
       " ",
       "{",
       indent([hardline, join(hardline, cases)]),
       hardline,
       "}"
     ]);
   }
   ```

3. **for内包表記の複雑なフォーマット**
   ```typescript
   // 複数の生成子とガード条件の整形
   function printForComprehension(
     enumerators: Doc[],
     body: Doc,
     hasYield: boolean
   ) {
     const delimiter = enumerators.length > 1 ? hardline : line;
     return group([
       "for",
       " ",
       enumerators.length > 1 ? "{" : "(",
       indent([delimiter, join([";", line], enumerators)]),
       delimiter,
       enumerators.length > 1 ? "}" : ")",
       hasYield ? [line, "yield"] : "",
       line,
       body
     ]);
   }
   ```

4. **文字列補間の処理**
   ```typescript
   // s"Hello ${name}"形式の処理
   function printStringInterpolation(
     prefix: string,
     parts: Array<string | Doc>
   ) {
     return [prefix, '"', ...parts, '"'];
   }
   ```

これらのプリンター実装パターンを理解することで、Scalaの複雑な構文に対応した高品質なフォーマッターを実装できます。