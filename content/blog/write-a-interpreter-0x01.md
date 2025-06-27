+++
date = '2025-03-31T15:25:18+08:00'
draft = true
title = '写个脚本解释器吧（一）'
+++

## 前言

上班苦闷，不如来写个脚本解释器来调剂一下生活吧。

*前排警告：非专业人员，文章涉及的术语以及做法不严谨，仅供参考，不建议直接模仿。*

## 目标

脚本解释器，一般是解析并运行一段代码。在这系列文章中，我们是准备达成这个目标，能够跑下面的代码。

```cpp
std::string script = "fn add(a, b) { return a + b; } return add(1, 1);"
auto ret = eval(script);
assert!(ret == 2 );
```

为了实现这个目标，我们需要做一些任务分解：

1. 把我们的脚本解析成AST。

2. 运行这个AST。


本篇文章先来做第一步，解析文本，生成AST。

## 文本解析

### 语法

为了我们的目标，我们先来一个大的，定义一个脚本语言，它的语法如下：

1. 基本类型

    boolean: 布尔类型。`true`, `false`。

    integer: 整型数，从-2^23到+2^23。`123`。

    float: 浮点数，double类型。`1.23`。

    string: 字符串，双引号包住字符串。`"hello world"`。

2. 语句

    我们的脚本语言主要是一个或多个语句组成，简单起见，我们把它们分为以下几种：

    1. 表达式语句：`1 + 1;`。

    2. 变量声明：`let x; let x = 1;`。

    3. 控制流：`if (x > 0) { x = 1; } else { x = 2; }`。

    4. 循环调用：`while (x < 10) { x = x + 1; }`。

    5. 函数声明：`fn add(a, b) { return a + b; }`。


### AST

AST，即抽象语法树，就是描述这个脚本语言的子元素构成。

**注意：以下的代码是伪代码，忽略很多细节，仅用于说明思路**

根据上面的语法，我们从上往下搞一个AST定义：

先有个ASTNode基类，然后根据语法定义，定义各种AST节点。

```cpp
class ASTNode {
    enum class Kind {
        Program,
        Statement,
        ExpressionStmt,
        LetStmt,
        AssignExpr,
        BinaryExpr,
        UnaryExpr,
        IdentifierExpr,
        LiteralExpr,
    }
public:
    virtual Kind kind();
}

```

#### 语句（Statement）

我们的脚本语言就是个Program，里面包含多个Statement。

```cpp
class Program: public ASTNode {
    Kind kind() override {}
    std::vector<Statement> statements;
}
```

而语句的话，就有很多不同的类型了。

```cpp
class Statement: public ASTNode {}
```

比如:

1. 空语句

只有一个`;`

```cpp
class EmptyStmt: public Statement {
    Kind kind() override { return Kind::EmptyStmt; }
}
```

2. 变量声明

`let x; let x = 1;`

```cpp
class LetStmt: public Statement {
    Kind kind() override { return Kind::LetStmt; }
    std::string name;
    std::unique_ptr<Expression> value;
}
```

3. 表达式语句

`1 + 1;`

```cpp
class ExpressionStmt: public Statement {
    Kind kind() override { return Kind::ExpressionStmt; }
    std::unique_ptr<Expression> expression;
}
```

4. 控制流

`if x > 0 { x = 1; } else { x = 2; }`

```cpp
class IfStmt: public Statement {
    Kind kind() override { return Kind::IfStmt; }
    std::unique_ptr<Expression> condition;
    std::unique_ptr<Statement> thenBranch;
    std::unique_ptr<Statement> elseBranch;
}
```

5. 循环调用

`while i < 10 { i = i + 1; }`

```cpp
class WhileStmt: public Statement {
    Kind kind() override { return Kind::WhileStmt; }
    std::unique_ptr<Expression> condition;
    std::unique_ptr<Statement> body;
}
```

#### 表达式（Expression）

另外，除了语句之外，脚本语言还有一个重要的组成，就是表达式。

```cpp
class Expression: public ASTNode {}
```

1. 二元表达式

`1 + 1`、 `1 >= 1`

```cpp
class BinaryExpr: public Expression {
    enum class Operator {
        Add, // +
        Sub, // -
        Mul, // *
        Div, // /
        Mod, // %
        Eq,  // ==
        Neq, // !=
        Gt,  // >
        Lt,  // <
        Gte, // >=
        Lte, // <=
    }

    Kind kind() override { return Kind::BinaryExpr; }
    Operator op;
    std::unique_ptr<Expression> left;
    std::unique_ptr<Expression> right;
}
```

2. 一元表达式

`!false`、`-1`

```cpp
class UnaryExpr: public Expression {
    enum class Operator {
        Neg, // -
        Not, // !
    }

    Kind kind() override { return Kind::UnaryExpr; }
    Operator op;
    std::unique_ptr<Expression> value;
}
```

3. 赋值表达式

`x = 1`

```cpp
class AssignExpr: public Expression {
    Kind kind() override { return Kind::AssignExpr; }
    std::unique_ptr<Expression> object;
    std::unique_ptr<Expression> value;
}
```

4. 标记符表达式

`x`、`a`。

```cpp
class IdentifierExpr: public Expression {
    Kind kind() override { return Kind::IdentifierExpr; }

    std::string name;
}
```

5. 字面量表达式

`1`、`"hello world"`、`true`、`false`。

```cpp
class LiteralExpr: public Expression {
    enum class LiteralKind {
        Boolean,
        Integer,
        Float,
        String,
    }

    Kind kind() override { return Kind::LiteralExpr; }

    virtual LiteralKind literalKind();
}

class BooleanLiteral: LiteralExpr {
    LiteralKind literalKind() override { return LiteralKind::Boolean; }

    bool value;
}

class IntegerLiteral: LiteralExpr {
    LiteralKind literalKind() override { return LiteralKind::Integer; }

    int64_t value;
}

class FloatLiteral: LiteralExpr {
    LiteralKind literalKind() override { return LiteralKind::Float; }

    double value;
}

class StringLiteral: LiteralExpr {
    LiteralKind literalKind() override { return LiteralKind::String; }

    std::string value;
}

```

6. 函数调用

`add(1, 2)`

```cpp
class CallExpr: public Expression {
    Kind kind() override { return Kind::CallExpr; }
    std::unique_ptr<Expression> callee;
    std::vector<std::unique_ptr<Expression>> arguments;
}
```

### Tokenizer

为了把一连串的字符解析为AST，我们先来搞一个Tokenizer。

Tokenizer就是把一个字符串，按照语法规则，切分成一个个Token。

比如 `let x = 1 + 1;`，在Tokenizer里，会切分成以下这些Token：

1. 关键字let： `let`
2. 标记符x：`x`
3. 标点符号=：`=`
4. 整型数1：`1`
5. 标点符号+：`+`
6. 整型数1：`1`。
7. 标点符号;：`;`

可以看到，对于空白字符，比如空格，Tab，回车，换行，我们直接忽略。而对于其它字符，比如字母，数字，符号，我们按照语法规则，切分成一个个Token。

```cpp
class Token {
    enum class Kind {
        Identifier, // 标记符
        Boolean,    // true or false
        Integer,    // 整型数
        Float,      // 浮点数
        String,     // 字符串
        LeftParen,  // (
        RightParen, // )
        LeftBrace,  // {
        RightBrace, // }
        Colon,      // :
        Comma,      // ,
        Semicolon,  // ;
    }

    Kind kind;
    std::string_view text;
}

class Tokenizer {
    char peek_char() {
        if (input.empty()) { return 0; }
    }
    char next_char() {}

    Token next_token() {}

    std::string_view input;
}
```