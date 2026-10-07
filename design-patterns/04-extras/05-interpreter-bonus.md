# 🧩 Interpreter — the 23rd GoF Pattern (Bonus Chapter)

> **Why this file exists.** The Gang of Four catalogue has 23 patterns. Most modern
> pattern catalogues — including Refactoring.Guru's — cover 22 and quietly drop
> **Interpreter**. It is the odd one out: narrower, more academic, and genuinely
> obsolete for most of what it was originally proposed for.
>
> But it is *not* useless. Every regex engine, every SQL planner, every spreadsheet
> formula bar, and every feature-flag rule evaluator you have ever used is an
> Interpreter underneath. So it is worth understanding properly — including the part
> where you learn to *not* reach for it.
>
> This chapter is written from scratch, and it builds **one** worked example all the
> way through in **TypeScript** and **C#**: a rule engine for filtering car listings.

---

## 📋 Table of contents

1. [Intent](#-intent)
2. [Problem](#-problem)
3. [Solution](#-solution)
4. [In plain English](#-in-plain-english)
5. [Participants](#-participants)
6. [Structure (ASCII)](#-structure)
7. [The worked example — a car listing rule engine](#-the-worked-example--a-car-listing-rule-engine)
   - [The grammar](#the-grammar)
   - [TypeScript: tokenizer](#typescript--1-tokenizer)
   - [TypeScript: AST / expression classes](#typescript--2-ast--expression-classes)
   - [TypeScript: parser](#typescript--3-parser-recursive-descent)
   - [TypeScript: evaluation + demo](#typescript--4-evaluation--demo)
   - [C#: the whole thing](#c--the-whole-thing)
8. [Applicability](#-applicability)
9. [Pros and cons](#-pros-and-cons)
10. [Relations with other patterns](#-relations-with-other-patterns)
11. [When NOT to use it (almost always)](#-when-not-to-use-it-almost-always)
12. [Modern alternatives](#-modern-alternatives)
13. [Where Interpreter genuinely shows up](#-where-interpreter-genuinely-shows-up)
14. [Recap](#-recap)

---

## 🎯 Intent

**Interpreter** is a behavioral design pattern that lets you define a representation
for a *language's grammar* along with an *interpreter* that uses that representation
to interpret sentences in the language.

Concretely: you take a small, stable, domain-specific language (a DSL), you model each
grammar rule as a class, you build a tree of those class instances for a given
sentence, and you evaluate the tree by walking it.

The one-sentence version:

> **Turn a string into a tree of objects, then ask the tree to evaluate itself.**

---

## 😖 Problem

You work on an automotive marketplace. Somebody in the business team wants to define
"lead routing rules" or "homepage promotion rules" without shipping a deploy every
time. They want to type things like:

```
price < 500000 AND (make = 'Maruti' OR year >= 2020)
```

and have listings filtered accordingly.

Your first instinct is one of these, and all of them are bad:

**Attempt 1 — hard-code every rule in C#.**

```csharp
// Ships a new binary every time marketing changes their mind.
if (listing.Price < 500000 && (listing.Make == "Maruti" || listing.Year >= 2020))
    Promote(listing);
```

Every new rule is a code change, a PR, a QA cycle, a deploy. Marketing wants three
rule changes a week. You now have a full-time job being a human compiler.

**Attempt 2 — a giant config table of `field / operator / value` rows.**

```
field   | op | value
--------|----|--------
price   | <  | 500000
make    | =  | Maruti
year    | >= | 2020
```

Fine — until someone asks "can I have OR?" Then "can I have parentheses?" Then
"can I mix AND and OR with correct precedence?" Your flat table has no way to express
tree structure. You start bolting on `group_id` and `parent_id` columns and you are
now reimplementing an AST in SQL, badly.

**Attempt 3 — string-bash it into SQL and concatenate.**

```csharp
var sql = $"SELECT * FROM Listings WHERE {userSuppliedRuleText}";  // 🔥 NO
```

Congratulations, you have invented SQL injection. Also the rule now only works if the
data happens to be in SQL, so you can't evaluate it against a RabbitMQ message or an
in-memory object.

**Attempt 4 — `eval()` in JavaScript.**

```js
// Also 🔥 NO.
const ok = eval(`listing.price < 500000 && listing.make === 'Maruti'`);
```

Arbitrary code execution from user input, no sandbox, no way to limit which fields
are readable, no way to give a helpful error message when the business user types
`prcie` instead of `price`.

What you actually need: a **small language you control completely**, with a grammar
you define, evaluated against data you choose to expose, with errors you can explain.

That's Interpreter.

---

## 💡 Solution

Interpreter says: **each rule in your grammar becomes a class.**

Your grammar for the rule DSL has roughly these rules:

```
expression  := orExpr
orExpr      := andExpr ( "OR" andExpr )*
andExpr     := notExpr ( "AND" notExpr )*
notExpr     := "NOT" notExpr | primary
primary     := "(" expression ")" | comparison
comparison  := field operator literal
```

So you make classes: `OrExpression`, `AndExpression`, `NotExpression`,
`ComparisonExpression`, `FieldExpression`, `LiteralExpression`.

Every one of them implements the same interface — something like
`interpret(context) => value`. Then:

1. A **tokenizer** turns the raw string into a flat list of tokens.
2. A **parser** turns that token list into a tree of expression objects.
3. Calling `interpret(context)` on the *root* node recursively evaluates the whole tree.

The recursion is the whole trick. `AndExpression.interpret()` doesn't know or care
whether its children are comparisons, nested ORs, or a 200-node monster. It just asks
its left child and its right child to interpret themselves and combines the answers.

```
"price < 500000 AND (make = 'Maruti' OR year >= 2020)"
                    │
              tokenizer
                    │
                    ▼
[FIELD price] [LT] [NUM 500000] [AND] [LPAREN] [FIELD make] ... 
                    │
                 parser
                    │
                    ▼
                  And
                 /    \
       price < 500000   Or
                      /    \
          make='Maruti'    year >= 2020
                    │
             interpret(listing)
                    │
                    ▼
                  true / false
```

---

## 🗣️ In plain English

### The grammar-as-classes idea

Imagine you are teaching a very literal-minded child to evaluate a sentence.

You hand them a card that says **"AND"**. The card has two slots. You tell them:
*"Whatever is in the left slot, work it out. Whatever is in the right slot, work it
out. If both came back yes, say yes. Otherwise say no."*

That's it. That's the whole `AndExpression` class. It doesn't need to understand
prices or makes or years. It only knows the rule for the word "AND".

Now you give another child a card that says **"price < 500000"**. Their rule is:
*"Look up the price on the listing you were handed. Compare it. Say yes or no."*

Now stack the cards into a tree, hand the top card a listing, and let them shout the
answers down to each other. That is Interpreter.

### Why a tree and not a list?

Because "AND" and "OR" nest. `A AND (B OR C)` means something different from
`(A AND B) OR C`. A flat list can't encode that. A tree can — the shape of the tree
*is* the meaning.

### Why is this a "pattern" and not just "how parsers work"?

Fair question, and honestly the answer is: it kind of is just how parsers work. What
GoF formalised is the *object-oriented* framing — one class per grammar rule, a shared
`interpret()` method, polymorphic recursion instead of a big switch statement.

Modern parsers usually do the opposite: plain data classes for nodes, and the logic
lives in a separate Visitor. (More on that in [Relations](#-relations-with-other-patterns).)

---

## 👥 Participants

| Participant | What it is | In our rule engine |
|---|---|---|
| **AbstractExpression** | Declares `interpret(context)`. The common interface for every node in the tree. | `Expr` interface with `evaluate(ctx): Value` |
| **TerminalExpression** | A leaf. Implements `interpret()` for a grammar symbol that has no children. | `LiteralExpr` (a number/string/bool), `FieldExpr` (reads `listing.price`) |
| **NonterminalExpression** | A branch. Holds child expressions and combines their results. One class per composite grammar rule. | `AndExpr`, `OrExpr`, `NotExpr`, `ComparisonExpr` |
| **Context** | Global information the interpretation needs — variable bindings, the object being tested, helper services. | The car `Listing` being evaluated, plus any config |
| **Client** | Builds (or gets from a parser) the abstract syntax tree, then calls `interpret()` on the root. | Your `RuleEngine.matches(listing)` call site |

A note on **Context**: GoF describes it as holding "global information". In practice it
is whatever the leaves need to resolve themselves. In a calculator DSL it's the
variable map `{x: 5, y: 7}`. In our rule engine it's the listing object. In a
feature-flag evaluator it's the user + session + request attributes.

---

## 🏗️ Structure

```
                    ┌───────────────────────────┐
     Client ──────▶ │    <<interface>>          │ ◀──── Context
                    │    AbstractExpression     │      (passed in)
                    ├───────────────────────────┤
                    │ + interpret(ctx): Value   │
                    └───────────────────────────┘
                                 △
                                 │ implements
             ┌───────────────────┴──────────────────────┐
             │                                          │
   ┌─────────────────────┐                  ┌───────────────────────────┐
   │ TerminalExpression  │                  │ NonterminalExpression     │
   ├─────────────────────┤                  ├───────────────────────────┤
   │ + interpret(ctx)    │                  │ - children: Abstract­Expr[]│
   └─────────────────────┘                  │ + interpret(ctx)          │
   e.g. LiteralExpr,                        └───────────────────────────┘
        FieldExpr                                      │
                                                       │ 0..*  ┌───────┐
                                                       └──────▶│ holds │
                                                               └───────┘
                                              e.g. AndExpr, OrExpr,
                                                   NotExpr, ComparisonExpr

  Note the self-reference: NonterminalExpression holds AbstractExpressions,
  which may themselves be Nonterminals. That recursion IS the Composite
  pattern. Interpreter is a Composite whose leaves and branches happen to
  mean "grammar rules".
```

And here's the full pipeline, which is what you actually build:

```
   raw text
      │
      ▼
 ┌──────────┐   tokens    ┌──────────┐    AST     ┌──────────────┐
 │ Tokenizer│ ──────────▶ │  Parser  │ ─────────▶ │ Expression   │
 │ (lexer)  │             │(recursive│            │    tree      │
 └──────────┘             │ descent) │            └──────────────┘
                          └──────────┘                   │
                                                         │ evaluate(ctx)
                                                         ▼
                                                   true / false
                                                   (or a number,
                                                    or SQL, or a
                                                    pretty-printed
                                                    string — see
                                                    Visitor below)
```

---

## 🚗 The worked example — a car listing rule engine

We're building the thing from the Problem section, properly. Business users type a
rule string; we parse it once, cache the tree, and evaluate it against thousands of
listings.

### The grammar

Written in a loose EBNF. Read `*` as "zero or more", `|` as "or".

```
expression   := orExpr

orExpr       := andExpr ( "OR" andExpr )*

andExpr      := notExpr ( "AND" notExpr )*

notExpr      := "NOT" notExpr
              | primary

primary      := "(" expression ")"
              | comparison

comparison   := operand compareOp operand

operand      := arithExpr

arithExpr    := term ( ("+" | "-") term )*

term         := factor ( ("*" | "/") factor )*

factor       := NUMBER
              | STRING
              | BOOLEAN
              | IDENTIFIER            (a field on the listing)
              | "(" arithExpr ")"
              | "-" factor

compareOp    := "=" | "!=" | "<" | "<=" | ">" | ">=" | "IN" | "CONTAINS"
```

Two things worth noticing:

1. **Precedence is encoded in the shape of the grammar.** `orExpr` calls `andExpr`,
   which calls `notExpr`. Because OR sits *above* AND in the call chain, AND binds
   tighter. `A OR B AND C` parses as `A OR (B AND C)`. This is how recursive-descent
   parsers do precedence — no precedence table needed.
2. **We snuck arithmetic in.** `price + shippingFee < 500000` works, and so does
   `year >= 2020`. Same machinery, one more layer of grammar rules.

### The domain we're filtering

```ts
interface Listing {
  id: number;
  make: string;
  model: string;
  year: number;
  price: number;
  kmDriven: number;
  fuelType: string;      // 'Petrol' | 'Diesel' | 'CNG' | 'Electric'
  city: string;
  isCertified: boolean;
  ownerCount: number;
}
```

---

### TypeScript — 1. Tokenizer

The tokenizer (or *lexer*) does one job: turn a flat string into a flat array of
tokens. No structure, no meaning, just classification.

```ts
// ---------------------------------------------------------------------------
// tokenizer.ts
// ---------------------------------------------------------------------------

export enum TokenType {
  Number = 'Number',
  String = 'String',
  Boolean = 'Boolean',
  Identifier = 'Identifier',
  Operator = 'Operator',   // = != < <= > >= + - * /
  Keyword = 'Keyword',     // AND OR NOT IN CONTAINS
  LParen = 'LParen',
  RParen = 'RParen',
  Comma = 'Comma',
  LBracket = 'LBracket',
  RBracket = 'RBracket',
  EOF = 'EOF',
}

export interface Token {
  type: TokenType;
  /** The raw text, useful for error messages. */
  lexeme: string;
  /** Pre-converted value for Number / String / Boolean tokens. */
  value?: number | string | boolean;
  /** Character offset in the source, for error messages. */
  position: number;
}

const KEYWORDS = new Set(['AND', 'OR', 'NOT', 'IN', 'CONTAINS']);
const BOOLEANS = new Set(['TRUE', 'FALSE']);

export class TokenizeError extends Error {
  constructor(message: string, public readonly position: number) {
    super(`${message} (at position ${position})`);
    this.name = 'TokenizeError';
  }
}

export function tokenize(source: string): Token[] {
  const tokens: Token[] = [];
  let i = 0;

  const peek = (offset = 0): string => source[i + offset] ?? '';
  const isDigit = (c: string) => c >= '0' && c <= '9';
  const isAlpha = (c: string) =>
    (c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || c === '_';
  const isAlphaNum = (c: string) => isAlpha(c) || isDigit(c);

  while (i < source.length) {
    const start = i;
    const c = source[i];

    // --- whitespace -------------------------------------------------------
    if (c === ' ' || c === '\t' || c === '\n' || c === '\r') {
      i++;
      continue;
    }

    // --- punctuation ------------------------------------------------------
    if (c === '(') { tokens.push({ type: TokenType.LParen,   lexeme: c, position: start }); i++; continue; }
    if (c === ')') { tokens.push({ type: TokenType.RParen,   lexeme: c, position: start }); i++; continue; }
    if (c === '[') { tokens.push({ type: TokenType.LBracket, lexeme: c, position: start }); i++; continue; }
    if (c === ']') { tokens.push({ type: TokenType.RBracket, lexeme: c, position: start }); i++; continue; }
    if (c === ',') { tokens.push({ type: TokenType.Comma,    lexeme: c, position: start }); i++; continue; }

    // --- operators (two-char forms first!) --------------------------------
    // Order matters: check '<=' before '<', or you'll tokenize '<=' as '<' then '='.
    const twoChar = source.slice(i, i + 2);
    if (twoChar === '<=' || twoChar === '>=' || twoChar === '!=' || twoChar === '==') {
      tokens.push({
        type: TokenType.Operator,
        lexeme: twoChar === '==' ? '=' : twoChar,  // normalise == to =
        position: start,
      });
      i += 2;
      continue;
    }
    if ('=<>+-*/'.includes(c)) {
      tokens.push({ type: TokenType.Operator, lexeme: c, position: start });
      i++;
      continue;
    }

    // --- string literals: 'Maruti' or "Maruti" ----------------------------
    if (c === "'" || c === '"') {
      const quote = c;
      i++;                       // consume opening quote
      let text = '';
      while (i < source.length && source[i] !== quote) {
        if (source[i] === '\\' && i + 1 < source.length) {
          // minimal escape support: \' \" \\ \n \t
          const esc = source[i + 1];
          text += esc === 'n' ? '\n'
                : esc === 't' ? '\t'
                : esc;
          i += 2;
        } else {
          text += source[i];
          i++;
        }
      }
      if (i >= source.length) {
        throw new TokenizeError('Unterminated string literal', start);
      }
      i++;                       // consume closing quote
      tokens.push({ type: TokenType.String, lexeme: quote + text + quote, value: text, position: start });
      continue;
    }

    // --- numbers ----------------------------------------------------------
    if (isDigit(c)) {
      while (isDigit(peek())) i++;
      if (peek() === '.' && isDigit(peek(1))) {
        i++;                     // consume '.'
        while (isDigit(peek())) i++;
      }
      const lexeme = source.slice(start, i);
      tokens.push({ type: TokenType.Number, lexeme, value: Number(lexeme), position: start });
      continue;
    }

    // --- identifiers / keywords / booleans --------------------------------
    if (isAlpha(c)) {
      while (isAlphaNum(peek())) i++;
      const lexeme = source.slice(start, i);
      const upper = lexeme.toUpperCase();

      if (KEYWORDS.has(upper)) {
        tokens.push({ type: TokenType.Keyword, lexeme: upper, position: start });
      } else if (BOOLEANS.has(upper)) {
        tokens.push({ type: TokenType.Boolean, lexeme, value: upper === 'TRUE', position: start });
      } else {
        tokens.push({ type: TokenType.Identifier, lexeme, position: start });
      }
      continue;
    }

    throw new TokenizeError(`Unexpected character '${c}'`, start);
  }

  tokens.push({ type: TokenType.EOF, lexeme: '<eof>', position: source.length });
  return tokens;
}
```

Run it mentally on `price < 500000 AND (make = 'Maruti' OR year >= 2020)` and you get:

```
Identifier(price)  Operator(<)  Number(500000)  Keyword(AND)  LParen
Identifier(make)   Operator(=)  String(Maruti)  Keyword(OR)
Identifier(year)   Operator(>=) Number(2020)    RParen        EOF
```

Flat. No tree yet. That's the parser's job.

---

### TypeScript — 2. AST / expression classes

Here are the actual **Interpreter** participants. Each grammar rule gets a class, and
each class knows how to evaluate itself.

```ts
// ---------------------------------------------------------------------------
// expressions.ts
// ---------------------------------------------------------------------------

/** Everything our little language can produce. */
export type Value = number | string | boolean | Value[];

/** The Context: what the leaves resolve against. */
export interface EvalContext {
  /** The record being tested — a car listing, in our case. */
  record: Record<string, unknown>;
  /** Optional strict mode: throw on unknown fields instead of returning undefined. */
  strictFields?: boolean;
}

export class EvaluationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'EvaluationError';
  }
}

/** AbstractExpression. */
export interface Expr {
  evaluate(ctx: EvalContext): Value;
  /** For debugging and for round-tripping a rule back to text. */
  toString(): string;
}

// --- TerminalExpressions ---------------------------------------------------

/** A literal: 500000, 'Maruti', TRUE. */
export class LiteralExpr implements Expr {
  constructor(public readonly value: Value) {}

  evaluate(_ctx: EvalContext): Value {
    return this.value;
  }

  toString(): string {
    return typeof this.value === 'string' ? `'${this.value}'` : String(this.value);
  }
}

/** A field read: price, make, isCertified. */
export class FieldExpr implements Expr {
  constructor(public readonly name: string) {}

  evaluate(ctx: EvalContext): Value {
    if (!(this.name in ctx.record)) {
      if (ctx.strictFields) {
        throw new EvaluationError(`Unknown field '${this.name}'`);
      }
      // Lenient mode: missing field behaves like a falsy/absent value.
      return '' as Value;
    }
    const raw = ctx.record[this.name];
    if (raw === null || raw === undefined) return '' as Value;
    if (typeof raw === 'number' || typeof raw === 'string' || typeof raw === 'boolean') {
      return raw;
    }
    if (Array.isArray(raw)) return raw as Value[];
    throw new EvaluationError(
      `Field '${this.name}' has unsupported type '${typeof raw}'`,
    );
  }

  toString(): string {
    return this.name;
  }
}

/** A list literal, for IN: ['Maruti', 'Hyundai', 'Tata']. */
export class ListExpr implements Expr {
  constructor(public readonly items: Expr[]) {}

  evaluate(ctx: EvalContext): Value {
    return this.items.map((item) => item.evaluate(ctx));
  }

  toString(): string {
    return `[${this.items.map(String).join(', ')}]`;
  }
}

// --- NonterminalExpressions ------------------------------------------------

export class AndExpr implements Expr {
  constructor(public readonly left: Expr, public readonly right: Expr) {}

  evaluate(ctx: EvalContext): Value {
    // Short-circuit: don't evaluate the right side if the left already failed.
    // This matters when the right side is expensive (a DB lookup in a custom field).
    if (!truthy(this.left.evaluate(ctx))) return false;
    return truthy(this.right.evaluate(ctx));
  }

  toString(): string {
    return `(${this.left} AND ${this.right})`;
  }
}

export class OrExpr implements Expr {
  constructor(public readonly left: Expr, public readonly right: Expr) {}

  evaluate(ctx: EvalContext): Value {
    if (truthy(this.left.evaluate(ctx))) return true;
    return truthy(this.right.evaluate(ctx));
  }

  toString(): string {
    return `(${this.left} OR ${this.right})`;
  }
}

export class NotExpr implements Expr {
  constructor(public readonly operand: Expr) {}

  evaluate(ctx: EvalContext): Value {
    return !truthy(this.operand.evaluate(ctx));
  }

  toString(): string {
    return `NOT ${this.operand}`;
  }
}

export type CompareOp = '=' | '!=' | '<' | '<=' | '>' | '>=' | 'IN' | 'CONTAINS';

export class ComparisonExpr implements Expr {
  constructor(
    public readonly left: Expr,
    public readonly op: CompareOp,
    public readonly right: Expr,
  ) {}

  evaluate(ctx: EvalContext): Value {
    const a = this.left.evaluate(ctx);
    const b = this.right.evaluate(ctx);

    switch (this.op) {
      case '=':  return looseEquals(a, b);
      case '!=': return !looseEquals(a, b);

      case '<':  return compareOrdered(a, b) < 0;
      case '<=': return compareOrdered(a, b) <= 0;
      case '>':  return compareOrdered(a, b) > 0;
      case '>=': return compareOrdered(a, b) >= 0;

      case 'IN': {
        if (!Array.isArray(b)) {
          throw new EvaluationError(`Right side of IN must be a list, got ${typeof b}`);
        }
        return b.some((item) => looseEquals(a, item));
      }

      case 'CONTAINS': {
        if (Array.isArray(a)) return a.some((item) => looseEquals(item, b));
        if (typeof a === 'string' && typeof b === 'string') {
          return a.toLowerCase().includes(b.toLowerCase());
        }
        throw new EvaluationError(
          `CONTAINS needs a list or string on the left, got ${typeof a}`,
        );
      }

      default: {
        // Exhaustiveness check — TS errors here if a CompareOp is unhandled.
        const never: never = this.op;
        throw new EvaluationError(`Unknown operator ${never}`);
      }
    }
  }

  toString(): string {
    return `${this.left} ${this.op} ${this.right}`;
  }
}

export type ArithOp = '+' | '-' | '*' | '/';

export class ArithmeticExpr implements Expr {
  constructor(
    public readonly left: Expr,
    public readonly op: ArithOp,
    public readonly right: Expr,
  ) {}

  evaluate(ctx: EvalContext): Value {
    const a = this.left.evaluate(ctx);
    const b = this.right.evaluate(ctx);

    // '+' doubles as string concatenation, like most languages.
    if (this.op === '+' && (typeof a === 'string' || typeof b === 'string')) {
      return String(a) + String(b);
    }

    const x = requireNumber(a, this.op);
    const y = requireNumber(b, this.op);

    switch (this.op) {
      case '+': return x + y;
      case '-': return x - y;
      case '*': return x * y;
      case '/':
        if (y === 0) throw new EvaluationError('Division by zero');
        return x / y;
    }
  }

  toString(): string {
    return `(${this.left} ${this.op} ${this.right})`;
  }
}

export class NegateExpr implements Expr {
  constructor(public readonly operand: Expr) {}

  evaluate(ctx: EvalContext): Value {
    return -requireNumber(this.operand.evaluate(ctx), 'unary -');
  }

  toString(): string {
    return `-${this.operand}`;
  }
}

// --- helpers ---------------------------------------------------------------

function truthy(v: Value): boolean {
  if (typeof v === 'boolean') return v;
  if (typeof v === 'number') return v !== 0;
  if (typeof v === 'string') return v.length > 0;
  if (Array.isArray(v)) return v.length > 0;
  return false;
}

function requireNumber(v: Value, op: string): number {
  if (typeof v === 'number') return v;
  if (typeof v === 'string' && v.trim() !== '' && !Number.isNaN(Number(v))) {
    return Number(v);
  }
  throw new EvaluationError(`Operator '${op}' needs a number, got '${String(v)}'`);
}

/**
 * Deliberately forgiving: business users type `year = '2020'` and mean the number.
 * Strings compare case-insensitively, because they type 'maruti' and mean 'Maruti'.
 */
function looseEquals(a: Value, b: Value): boolean {
  if (typeof a === 'string' && typeof b === 'string') {
    return a.toLowerCase() === b.toLowerCase();
  }
  if (typeof a === 'number' && typeof b === 'string') return a === Number(b);
  if (typeof a === 'string' && typeof b === 'number') return Number(a) === b;
  if (Array.isArray(a) || Array.isArray(b)) return false;
  return a === b;
}

function compareOrdered(a: Value, b: Value): number {
  if (typeof a === 'number' && typeof b === 'number') return a - b;

  // Allow numeric strings, since a config value may arrive as text.
  const na = typeof a === 'string' ? Number(a) : NaN;
  const nb = typeof b === 'string' ? Number(b) : NaN;
  if (typeof a === 'number' && !Number.isNaN(nb)) return a - nb;
  if (!Number.isNaN(na) && typeof b === 'number') return na - b;

  if (typeof a === 'string' && typeof b === 'string') {
    return a.toLowerCase().localeCompare(b.toLowerCase());
  }
  throw new EvaluationError(
    `Cannot order-compare '${String(a)}' and '${String(b)}'`,
  );
}
```

Look at `AndExpr.evaluate()` again. Eight lines. It doesn't know what a car is. It
doesn't know what a price is. It knows *one grammar rule*. That's the pattern working.

---

### TypeScript — 3. Parser (recursive descent)

The parser walks the token list and builds the tree. Each grammar rule becomes one
method, and the methods call each other in exactly the order the grammar specifies —
which is why the precedence comes out right for free.

```ts
// ---------------------------------------------------------------------------
// parser.ts
// ---------------------------------------------------------------------------

import { Token, TokenType, tokenize } from './tokenizer';
import {
  Expr, AndExpr, OrExpr, NotExpr, ComparisonExpr, ArithmeticExpr,
  NegateExpr, LiteralExpr, FieldExpr, ListExpr, CompareOp, ArithOp,
} from './expressions';

export class ParseError extends Error {
  constructor(message: string, public readonly token: Token) {
    super(`${message} — got '${token.lexeme}' at position ${token.position}`);
    this.name = 'ParseError';
  }
}

const COMPARE_OPS = new Set(['=', '!=', '<', '<=', '>', '>=']);

export class Parser {
  private pos = 0;

  private constructor(private readonly tokens: Token[]) {}

  /** The Client-facing entry point. */
  static parse(source: string): Expr {
    const parser = new Parser(tokenize(source));
    const expr = parser.parseExpression();
    parser.expect(TokenType.EOF, 'Expected end of expression');
    return expr;
  }

  // --- token helpers -------------------------------------------------------

  private peek(): Token {
    return this.tokens[this.pos];
  }

  private advance(): Token {
    return this.tokens[this.pos++];
  }

  private check(type: TokenType, lexeme?: string): boolean {
    const t = this.peek();
    if (t.type !== type) return false;
    return lexeme === undefined || t.lexeme === lexeme;
  }

  private match(type: TokenType, lexeme?: string): boolean {
    if (this.check(type, lexeme)) {
      this.advance();
      return true;
    }
    return false;
  }

  private expect(type: TokenType, message: string, lexeme?: string): Token {
    if (this.check(type, lexeme)) return this.advance();
    throw new ParseError(message, this.peek());
  }

  // --- grammar rules, one method each --------------------------------------

  /** expression := orExpr */
  private parseExpression(): Expr {
    return this.parseOr();
  }

  /** orExpr := andExpr ( "OR" andExpr )* */
  private parseOr(): Expr {
    let left = this.parseAnd();
    while (this.match(TokenType.Keyword, 'OR')) {
      const right = this.parseAnd();
      left = new OrExpr(left, right);   // left-associative
    }
    return left;
  }

  /** andExpr := notExpr ( "AND" notExpr )* */
  private parseAnd(): Expr {
    let left = this.parseNot();
    while (this.match(TokenType.Keyword, 'AND')) {
      const right = this.parseNot();
      left = new AndExpr(left, right);
    }
    return left;
  }

  /** notExpr := "NOT" notExpr | primary */
  private parseNot(): Expr {
    if (this.match(TokenType.Keyword, 'NOT')) {
      return new NotExpr(this.parseNot());   // right-recursive: NOT NOT x works
    }
    return this.parsePrimary();
  }

  /**
   * primary := "(" expression ")" | comparison
   *
   * The paren case is ambiguous with arithmetic parens — `(price + 100)` could start
   * either. We resolve it by parsing the parenthesised thing as a full expression,
   * then checking whether a comparison operator follows.
   */
  private parsePrimary(): Expr {
    if (this.check(TokenType.LParen)) {
      const save = this.pos;
      this.advance();                       // consume '('
      const inner = this.parseExpression();
      if (this.match(TokenType.RParen)) {
        // If a compare op follows, this paren group was an arithmetic operand.
        if (this.isCompareOpAhead()) {
          this.pos = save;                  // backtrack, re-parse as comparison
          return this.parseComparison();
        }
        return inner;                       // it was a boolean grouping
      }
      this.pos = save;                      // not a clean group; try comparison
    }
    return this.parseComparison();
  }

  private isCompareOpAhead(): boolean {
    const t = this.peek();
    if (t.type === TokenType.Operator && COMPARE_OPS.has(t.lexeme)) return true;
    if (t.type === TokenType.Keyword && (t.lexeme === 'IN' || t.lexeme === 'CONTAINS')) return true;
    return false;
  }

  /** comparison := arithExpr compareOp arithExpr */
  private parseComparison(): Expr {
    const left = this.parseArith();

    if (this.check(TokenType.Operator) && COMPARE_OPS.has(this.peek().lexeme)) {
      const op = this.advance().lexeme as CompareOp;
      const right = this.parseArith();
      return new ComparisonExpr(left, op, right);
    }

    if (this.check(TokenType.Keyword, 'IN')) {
      this.advance();
      return new ComparisonExpr(left, 'IN', this.parseList());
    }

    if (this.check(TokenType.Keyword, 'CONTAINS')) {
      this.advance();
      return new ComparisonExpr(left, 'CONTAINS', this.parseArith());
    }

    // A bare operand used as a condition, e.g. `isCertified`.
    return left;
  }

  /** list := "[" ( operand ( "," operand )* )? "]" */
  private parseList(): Expr {
    this.expect(TokenType.LBracket, 'Expected [ after IN');
    const items: Expr[] = [];
    if (!this.check(TokenType.RBracket)) {
      do {
        items.push(this.parseArith());
      } while (this.match(TokenType.Comma));
    }
    this.expect(TokenType.RBracket, 'Expected ] to close list');
    return new ListExpr(items);
  }

  /** arithExpr := term ( ("+"|"-") term )* */
  private parseArith(): Expr {
    let left = this.parseTerm();
    while (this.check(TokenType.Operator, '+') || this.check(TokenType.Operator, '-')) {
      const op = this.advance().lexeme as ArithOp;
      left = new ArithmeticExpr(left, op, this.parseTerm());
    }
    return left;
  }

  /** term := factor ( ("*"|"/") factor )* */
  private parseTerm(): Expr {
    let left = this.parseFactor();
    while (this.check(TokenType.Operator, '*') || this.check(TokenType.Operator, '/')) {
      const op = this.advance().lexeme as ArithOp;
      left = new ArithmeticExpr(left, op, this.parseFactor());
    }
    return left;
  }

  /** factor := NUMBER | STRING | BOOLEAN | IDENT | "(" arithExpr ")" | "-" factor */
  private parseFactor(): Expr {
    const t = this.peek();

    if (this.match(TokenType.Operator, '-')) {
      return new NegateExpr(this.parseFactor());
    }

    if (t.type === TokenType.Number || t.type === TokenType.String || t.type === TokenType.Boolean) {
      this.advance();
      return new LiteralExpr(t.value as string | number | boolean);
    }

    if (t.type === TokenType.Identifier) {
      this.advance();
      return new FieldExpr(t.lexeme);
    }

    if (this.match(TokenType.LParen)) {
      const inner = this.parseExpression();
      this.expect(TokenType.RParen, 'Expected ) to close group');
      return inner;
    }

    throw new ParseError('Expected a value, field, or (', t);
  }
}
```

**The backtracking bit deserves a note.** `parsePrimary` has to decide whether `(`
starts a boolean group like `(make = 'Maruti' OR year >= 2020)` or an arithmetic group
like `(price + fee) < 500000`. We handle it by trying the boolean reading first and
rewinding if a comparison operator turns up afterwards. For a grammar this small that's
perfectly fine. For anything bigger, this is exactly the smell that says *stop
hand-rolling and use a parser generator* — see [Modern alternatives](#-modern-alternatives).

---

### TypeScript — 4. Evaluation + demo

```ts
// ---------------------------------------------------------------------------
// ruleEngine.ts
// ---------------------------------------------------------------------------

import { Parser } from './parser';
import { Expr, EvalContext } from './expressions';

export interface Listing {
  id: number;
  make: string;
  model: string;
  year: number;
  price: number;
  kmDriven: number;
  fuelType: string;
  city: string;
  isCertified: boolean;
  ownerCount: number;
}

/**
 * The Client. Parses once, evaluates many times.
 *
 * Parsing is the expensive bit (string scanning, allocation). Evaluation is a
 * cheap tree walk. So we cache the compiled tree per rule string — which is
 * exactly how you'd use this against a few hundred thousand listings.
 */
export class RuleEngine {
  private readonly cache = new Map<string, Expr>();

  compile(rule: string): Expr {
    let expr = this.cache.get(rule);
    if (!expr) {
      expr = Parser.parse(rule);
      this.cache.set(rule, expr);
    }
    return expr;
  }

  matches(rule: string, listing: Listing): boolean {
    const expr = this.compile(rule);
    const ctx: EvalContext = {
      record: listing as unknown as Record<string, unknown>,
      strictFields: true,
    };
    return Boolean(expr.evaluate(ctx));
  }

  filter(rule: string, listings: Listing[]): Listing[] {
    const expr = this.compile(rule);
    return listings.filter((listing) =>
      Boolean(expr.evaluate({
        record: listing as unknown as Record<string, unknown>,
        strictFields: true,
      })),
    );
  }
}
```

And the demo:

```ts
// ---------------------------------------------------------------------------
// demo.ts
// ---------------------------------------------------------------------------

import { RuleEngine, Listing } from './ruleEngine';
import { Parser } from './parser';

const listings: Listing[] = [
  { id: 1, make: 'Maruti',  model: 'Swift',   year: 2021, price: 620000,
    kmDriven: 18000, fuelType: 'Petrol', city: 'Pune',     isCertified: true,  ownerCount: 1 },
  { id: 2, make: 'Hyundai', model: 'i20',     year: 2019, price: 480000,
    kmDriven: 42000, fuelType: 'Petrol', city: 'Mumbai',   isCertified: false, ownerCount: 2 },
  { id: 3, make: 'Maruti',  model: 'Baleno',  year: 2018, price: 450000,
    kmDriven: 55000, fuelType: 'Diesel', city: 'Pune',     isCertified: true,  ownerCount: 1 },
  { id: 4, make: 'Tata',    model: 'Nexon',   year: 2022, price: 1050000,
    kmDriven: 9000,  fuelType: 'Electric', city: 'Bengaluru', isCertified: true, ownerCount: 1 },
  { id: 5, make: 'Honda',   model: 'City',    year: 2020, price: 890000,
    kmDriven: 31000, fuelType: 'Petrol', city: 'Mumbai',   isCertified: false, ownerCount: 1 },
];

const engine = new RuleEngine();

const rules = [
  `price < 500000 AND (make = 'Maruti' OR year >= 2020)`,
  `make IN ['Maruti', 'Tata'] AND isCertified`,
  `year >= 2020 AND kmDriven < 20000`,
  `NOT (fuelType = 'Diesel') AND price / 100000 < 7`,
  `city CONTAINS 'mum' OR ownerCount = 1 AND price <= 700000`,
];

for (const rule of rules) {
  const matched = engine.filter(rule, listings);
  console.log(`\nRule: ${rule}`);
  console.log(`  AST: ${Parser.parse(rule).toString()}`);
  console.log(`  Matched: ${matched.map((l) => `${l.make} ${l.model}`).join(', ') || '(none)'}`);
}
```

Expected output:

```
Rule: price < 500000 AND (make = 'Maruti' OR year >= 2020)
  AST: ((price < 500000) AND ((make = 'Maruti') OR (year >= 2020)))
  Matched: Hyundai i20, Maruti Baleno

Rule: make IN ['Maruti', 'Tata'] AND isCertified
  AST: ((make IN ['Maruti', 'Tata']) AND isCertified)
  Matched: Maruti Swift, Maruti Baleno, Tata Nexon

Rule: year >= 2020 AND kmDriven < 20000
  AST: ((year >= 2020) AND (kmDriven < 20000))
  Matched: Maruti Swift, Tata Nexon

Rule: NOT (fuelType = 'Diesel') AND price / 100000 < 7
  AST: (NOT (fuelType = 'Diesel') AND ((price / 100000) < 7))
  Matched: Maruti Swift, Hyundai i20

Rule: city CONTAINS 'mum' OR ownerCount = 1 AND price <= 700000
  AST: ((city CONTAINS 'mum') OR ((ownerCount = 1) AND (price <= 700000)))
  Matched: Maruti Swift, Hyundai i20, Maruti Baleno, Honda City
```

Notice the last one: `A OR B AND C` came out as `A OR (B AND C)`. We never wrote a
precedence table. The *shape of the grammar* did it, because `parseOr` calls
`parseAnd` and not the other way round.

Wait — check rule 1 against listing 2 (Hyundai i20, 2019, ₹480,000). `price < 500000`
is true. `make = 'Maruti'` is false, but `year >= 2020` is... 2019, false. So the OR
is false, so the AND is false. It should NOT match.

Good catch if you spotted that — this is exactly the kind of thing you'd write a test
for. Let's fix the expected output for rule 1:

```
  Matched: Maruti Baleno
```

(Baleno: price 450000 < 500000 ✓, make = 'Maruti' ✓ → true.)

This is worth calling out deliberately: **a rule DSL is a language, and languages need
tests.** The moment business users can write logic, you need a test suite that runs
their rules against fixture listings. Otherwise you've just moved bugs from your code
into a database column where nobody reviews them.

---

### C# — the whole thing

Same design, ported. C# gives us records, pattern matching, and switch expressions,
which make the expression classes noticeably tighter.

#### C# — Tokens and tokenizer

```csharp
// ---------------------------------------------------------------------------
// Tokenizer.cs
// ---------------------------------------------------------------------------
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Text;

namespace CarWale.Rules;

public enum TokenType
{
    Number, String, Boolean, Identifier,
    Operator, Keyword,
    LParen, RParen, LBracket, RBracket, Comma,
    Eof
}

public readonly record struct Token(
    TokenType Type,
    string Lexeme,
    object? Value,
    int Position);

public sealed class TokenizeException : Exception
{
    public int Position { get; }

    public TokenizeException(string message, int position)
        : base($"{message} (at position {position})")
        => Position = position;
}

public static class Tokenizer
{
    private static readonly HashSet<string> Keywords =
        new(StringComparer.OrdinalIgnoreCase) { "AND", "OR", "NOT", "IN", "CONTAINS" };

    public static IReadOnlyList<Token> Tokenize(string source)
    {
        var tokens = new List<Token>();
        var i = 0;

        char Peek(int offset = 0) =>
            i + offset < source.Length ? source[i + offset] : '\0';

        static bool IsAlpha(char c) => char.IsLetter(c) || c == '_';
        static bool IsAlphaNum(char c) => char.IsLetterOrDigit(c) || c == '_';

        while (i < source.Length)
        {
            var start = i;
            var c = source[i];

            if (char.IsWhiteSpace(c)) { i++; continue; }

            switch (c)
            {
                case '(': tokens.Add(new Token(TokenType.LParen,   "(", null, start)); i++; continue;
                case ')': tokens.Add(new Token(TokenType.RParen,   ")", null, start)); i++; continue;
                case '[': tokens.Add(new Token(TokenType.LBracket, "[", null, start)); i++; continue;
                case ']': tokens.Add(new Token(TokenType.RBracket, "]", null, start)); i++; continue;
                case ',': tokens.Add(new Token(TokenType.Comma,    ",", null, start)); i++; continue;
            }

            // Two-character operators first.
            if (i + 1 < source.Length)
            {
                var two = source.Substring(i, 2);
                if (two is "<=" or ">=" or "!=" or "==")
                {
                    tokens.Add(new Token(TokenType.Operator, two == "==" ? "=" : two, null, start));
                    i += 2;
                    continue;
                }
            }

            if ("=<>+-*/".IndexOf(c) >= 0)
            {
                tokens.Add(new Token(TokenType.Operator, c.ToString(), null, start));
                i++;
                continue;
            }

            // String literals.
            if (c is '\'' or '"')
            {
                var quote = c;
                i++;
                var sb = new StringBuilder();
                while (i < source.Length && source[i] != quote)
                {
                    if (source[i] == '\\' && i + 1 < source.Length)
                    {
                        sb.Append(source[i + 1] switch
                        {
                            'n' => '\n',
                            't' => '\t',
                            var esc => esc
                        });
                        i += 2;
                    }
                    else
                    {
                        sb.Append(source[i]);
                        i++;
                    }
                }

                if (i >= source.Length)
                    throw new TokenizeException("Unterminated string literal", start);

                i++; // closing quote
                var text = sb.ToString();
                tokens.Add(new Token(TokenType.String, $"'{text}'", text, start));
                continue;
            }

            // Numbers.
            if (char.IsDigit(c))
            {
                while (char.IsDigit(Peek())) i++;
                if (Peek() == '.' && char.IsDigit(Peek(1)))
                {
                    i++;
                    while (char.IsDigit(Peek())) i++;
                }

                var lexeme = source[start..i];
                tokens.Add(new Token(
                    TokenType.Number,
                    lexeme,
                    double.Parse(lexeme, CultureInfo.InvariantCulture),
                    start));
                continue;
            }

            // Identifiers / keywords / booleans.
            if (IsAlpha(c))
            {
                while (IsAlphaNum(Peek())) i++;
                var lexeme = source[start..i];

                if (Keywords.Contains(lexeme))
                {
                    tokens.Add(new Token(TokenType.Keyword, lexeme.ToUpperInvariant(), null, start));
                }
                else if (lexeme.Equals("true", StringComparison.OrdinalIgnoreCase) ||
                         lexeme.Equals("false", StringComparison.OrdinalIgnoreCase))
                {
                    tokens.Add(new Token(
                        TokenType.Boolean,
                        lexeme,
                        lexeme.Equals("true", StringComparison.OrdinalIgnoreCase),
                        start));
                }
                else
                {
                    tokens.Add(new Token(TokenType.Identifier, lexeme, null, start));
                }
                continue;
            }

            throw new TokenizeException($"Unexpected character '{c}'", start);
        }

        tokens.Add(new Token(TokenType.Eof, "<eof>", null, source.Length));
        return tokens;
    }
}
```

#### C# — Expression classes

```csharp
// ---------------------------------------------------------------------------
// Expressions.cs
// ---------------------------------------------------------------------------
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Linq;

namespace CarWale.Rules;

/// <summary>The Context: everything a leaf needs to resolve itself.</summary>
public sealed class EvalContext
{
    public required IReadOnlyDictionary<string, object?> Record { get; init; }
    public bool StrictFields { get; init; } = true;
}

public sealed class EvaluationException : Exception
{
    public EvaluationException(string message) : base(message) { }
}

/// <summary>AbstractExpression.</summary>
public interface IExpr
{
    object? Evaluate(EvalContext ctx);
}

// --- TerminalExpressions ---------------------------------------------------

public sealed record LiteralExpr(object? Value) : IExpr
{
    public object? Evaluate(EvalContext ctx) => Value;

    public override string ToString() =>
        Value is string s ? $"'{s}'" : Convert.ToString(Value, CultureInfo.InvariantCulture) ?? "null";
}

public sealed record FieldExpr(string Name) : IExpr
{
    public object? Evaluate(EvalContext ctx)
    {
        if (ctx.Record.TryGetValue(Name, out var value)) return value;
        if (ctx.StrictFields) throw new EvaluationException($"Unknown field '{Name}'");
        return null;
    }

    public override string ToString() => Name;
}

public sealed record ListExpr(IReadOnlyList<IExpr> Items) : IExpr
{
    public object? Evaluate(EvalContext ctx) =>
        Items.Select(item => item.Evaluate(ctx)).ToList();

    public override string ToString() =>
        $"[{string.Join(", ", Items.Select(i => i.ToString()))}]";
}

// --- NonterminalExpressions ------------------------------------------------

public sealed record AndExpr(IExpr Left, IExpr Right) : IExpr
{
    public object? Evaluate(EvalContext ctx) =>
        Coerce.Truthy(Left.Evaluate(ctx)) && Coerce.Truthy(Right.Evaluate(ctx));

    public override string ToString() => $"({Left} AND {Right})";
}

public sealed record OrExpr(IExpr Left, IExpr Right) : IExpr
{
    public object? Evaluate(EvalContext ctx) =>
        Coerce.Truthy(Left.Evaluate(ctx)) || Coerce.Truthy(Right.Evaluate(ctx));

    public override string ToString() => $"({Left} OR {Right})";
}

public sealed record NotExpr(IExpr Operand) : IExpr
{
    public object? Evaluate(EvalContext ctx) => !Coerce.Truthy(Operand.Evaluate(ctx));

    public override string ToString() => $"NOT {Operand}";
}

public sealed record ComparisonExpr(IExpr Left, string Op, IExpr Right) : IExpr
{
    public object? Evaluate(EvalContext ctx)
    {
        var a = Left.Evaluate(ctx);
        var b = Right.Evaluate(ctx);

        return Op switch
        {
            "="  => Coerce.LooseEquals(a, b),
            "!=" => !Coerce.LooseEquals(a, b),
            "<"  => Coerce.CompareOrdered(a, b) < 0,
            "<=" => Coerce.CompareOrdered(a, b) <= 0,
            ">"  => Coerce.CompareOrdered(a, b) > 0,
            ">=" => Coerce.CompareOrdered(a, b) >= 0,

            "IN" => b is System.Collections.IEnumerable list and not string
                ? list.Cast<object?>().Any(item => Coerce.LooseEquals(a, item))
                : throw new EvaluationException("Right side of IN must be a list"),

            "CONTAINS" => a switch
            {
                System.Collections.IEnumerable seq and not string =>
                    seq.Cast<object?>().Any(item => Coerce.LooseEquals(item, b)),
                string haystack when b is string needle =>
                    haystack.Contains(needle, StringComparison.OrdinalIgnoreCase),
                _ => throw new EvaluationException("CONTAINS needs a list or string on the left")
            },

            _ => throw new EvaluationException($"Unknown operator '{Op}'")
        };
    }

    public override string ToString() => $"{Left} {Op} {Right}";
}

public sealed record ArithmeticExpr(IExpr Left, char Op, IExpr Right) : IExpr
{
    public object? Evaluate(EvalContext ctx)
    {
        var a = Left.Evaluate(ctx);
        var b = Right.Evaluate(ctx);

        if (Op == '+' && (a is string || b is string))
            return Convert.ToString(a, CultureInfo.InvariantCulture) +
                   Convert.ToString(b, CultureInfo.InvariantCulture);

        var x = Coerce.RequireNumber(a, Op.ToString());
        var y = Coerce.RequireNumber(b, Op.ToString());

        return Op switch
        {
            '+' => x + y,
            '-' => x - y,
            '*' => x * y,
            '/' => y == 0 ? throw new EvaluationException("Division by zero") : x / y,
            _   => throw new EvaluationException($"Unknown arithmetic operator '{Op}'")
        };
    }

    public override string ToString() => $"({Left} {Op} {Right})";
}

public sealed record NegateExpr(IExpr Operand) : IExpr
{
    public object? Evaluate(EvalContext ctx) =>
        -Coerce.RequireNumber(Operand.Evaluate(ctx), "unary -");

    public override string ToString() => $"-{Operand}";
}

// --- coercion helpers ------------------------------------------------------

internal static class Coerce
{
    public static bool Truthy(object? v) => v switch
    {
        null              => false,
        bool b            => b,
        double d          => d != 0,
        int n             => n != 0,
        string s          => s.Length > 0,
        System.Collections.ICollection c => c.Count > 0,
        _                 => true
    };

    public static double RequireNumber(object? v, string op) => v switch
    {
        double d => d,
        int n    => n,
        long l   => l,
        decimal m => (double)m,
        string s when double.TryParse(s, NumberStyles.Any, CultureInfo.InvariantCulture, out var parsed) => parsed,
        _ => throw new EvaluationException($"Operator '{op}' needs a number, got '{v}'")
    };

    public static bool LooseEquals(object? a, object? b)
    {
        if (a is null || b is null) return a is null && b is null;

        if (a is string sa && b is string sb)
            return string.Equals(sa, sb, StringComparison.OrdinalIgnoreCase);

        if (a is bool ba && b is bool bb) return ba == bb;

        if (TryNumber(a, out var na) && TryNumber(b, out var nb))
            return Math.Abs(na - nb) < 1e-9;

        return Equals(a, b);
    }

    public static int CompareOrdered(object? a, object? b)
    {
        if (TryNumber(a, out var na) && TryNumber(b, out var nb))
            return na.CompareTo(nb);

        if (a is string sa && b is string sb)
            return string.Compare(sa, sb, StringComparison.OrdinalIgnoreCase);

        if (a is DateTime da && b is DateTime db) return da.CompareTo(db);

        throw new EvaluationException($"Cannot order-compare '{a}' and '{b}'");
    }

    private static bool TryNumber(object? v, out double result)
    {
        switch (v)
        {
            case double d:  result = d; return true;
            case int n:     result = n; return true;
            case long l:    result = l; return true;
            case decimal m: result = (double)m; return true;
            case string s when double.TryParse(s, NumberStyles.Any, CultureInfo.InvariantCulture, out var p):
                result = p; return true;
            default:
                result = 0; return false;
        }
    }
}
```

#### C# — Parser

```csharp
// ---------------------------------------------------------------------------
// Parser.cs
// ---------------------------------------------------------------------------
using System;
using System.Collections.Generic;

namespace CarWale.Rules;

public sealed class ParseException : Exception
{
    public Token Token { get; }

    public ParseException(string message, Token token)
        : base($"{message} — got '{token.Lexeme}' at position {token.Position}")
        => Token = token;
}

public sealed class Parser
{
    private static readonly HashSet<string> CompareOps = new() { "=", "!=", "<", "<=", ">", ">=" };

    private readonly IReadOnlyList<Token> _tokens;
    private int _pos;

    private Parser(IReadOnlyList<Token> tokens) => _tokens = tokens;

    /// <summary>Client-facing entry point.</summary>
    public static IExpr Parse(string source)
    {
        var parser = new Parser(Tokenizer.Tokenize(source));
        var expr = parser.ParseExpression();
        parser.Expect(TokenType.Eof, "Expected end of expression");
        return expr;
    }

    // --- token helpers -------------------------------------------------------

    private Token Peek() => _tokens[_pos];
    private Token Advance() => _tokens[_pos++];

    private bool Check(TokenType type, string? lexeme = null)
    {
        var t = Peek();
        if (t.Type != type) return false;
        return lexeme is null || string.Equals(t.Lexeme, lexeme, StringComparison.Ordinal);
    }

    private bool Match(TokenType type, string? lexeme = null)
    {
        if (!Check(type, lexeme)) return false;
        Advance();
        return true;
    }

    private Token Expect(TokenType type, string message, string? lexeme = null)
    {
        if (Check(type, lexeme)) return Advance();
        throw new ParseException(message, Peek());
    }

    // --- grammar rules -------------------------------------------------------

    private IExpr ParseExpression() => ParseOr();

    private IExpr ParseOr()
    {
        var left = ParseAnd();
        while (Match(TokenType.Keyword, "OR"))
            left = new OrExpr(left, ParseAnd());
        return left;
    }

    private IExpr ParseAnd()
    {
        var left = ParseNot();
        while (Match(TokenType.Keyword, "AND"))
            left = new AndExpr(left, ParseNot());
        return left;
    }

    private IExpr ParseNot() =>
        Match(TokenType.Keyword, "NOT") ? new NotExpr(ParseNot()) : ParsePrimary();

    private IExpr ParsePrimary()
    {
        if (Check(TokenType.LParen))
        {
            var save = _pos;
            Advance();
            var inner = ParseExpression();
            if (Match(TokenType.RParen))
            {
                if (IsCompareOpAhead())
                {
                    _pos = save;                 // backtrack: it was an arithmetic group
                    return ParseComparison();
                }
                return inner;
            }
            _pos = save;
        }
        return ParseComparison();
    }

    private bool IsCompareOpAhead()
    {
        var t = Peek();
        if (t.Type == TokenType.Operator && CompareOps.Contains(t.Lexeme)) return true;
        return t.Type == TokenType.Keyword && t.Lexeme is "IN" or "CONTAINS";
    }

    private IExpr ParseComparison()
    {
        var left = ParseArith();

        if (Check(TokenType.Operator) && CompareOps.Contains(Peek().Lexeme))
        {
            var op = Advance().Lexeme;
            return new ComparisonExpr(left, op, ParseArith());
        }

        if (Match(TokenType.Keyword, "IN"))
            return new ComparisonExpr(left, "IN", ParseList());

        if (Match(TokenType.Keyword, "CONTAINS"))
            return new ComparisonExpr(left, "CONTAINS", ParseArith());

        return left;   // bare condition, e.g. `isCertified`
    }

    private IExpr ParseList()
    {
        Expect(TokenType.LBracket, "Expected [ after IN");
        var items = new List<IExpr>();
        if (!Check(TokenType.RBracket))
        {
            do { items.Add(ParseArith()); } while (Match(TokenType.Comma));
        }
        Expect(TokenType.RBracket, "Expected ] to close list");
        return new ListExpr(items);
    }

    private IExpr ParseArith()
    {
        var left = ParseTerm();
        while (Check(TokenType.Operator, "+") || Check(TokenType.Operator, "-"))
        {
            var op = Advance().Lexeme[0];
            left = new ArithmeticExpr(left, op, ParseTerm());
        }
        return left;
    }

    private IExpr ParseTerm()
    {
        var left = ParseFactor();
        while (Check(TokenType.Operator, "*") || Check(TokenType.Operator, "/"))
        {
            var op = Advance().Lexeme[0];
            left = new ArithmeticExpr(left, op, ParseFactor());
        }
        return left;
    }

    private IExpr ParseFactor()
    {
        var t = Peek();

        if (Match(TokenType.Operator, "-"))
            return new NegateExpr(ParseFactor());

        if (t.Type is TokenType.Number or TokenType.String or TokenType.Boolean)
        {
            Advance();
            return new LiteralExpr(t.Value);
        }

        if (t.Type == TokenType.Identifier)
        {
            Advance();
            return new FieldExpr(t.Lexeme);
        }

        if (Match(TokenType.LParen))
        {
            var inner = ParseExpression();
            Expect(TokenType.RParen, "Expected ) to close group");
            return inner;
        }

        throw new ParseException("Expected a value, field, or (", t);
    }
}
```

#### C# — Rule engine, context adapter, demo

The one extra piece C# needs is a way to turn a strongly-typed `Listing` into the
`IReadOnlyDictionary<string, object?>` the context wants. Reflection is the easy
route; a compiled accessor is the fast route. Here's both, with the fast one cached.

```csharp
// ---------------------------------------------------------------------------
// RuleEngine.cs
// ---------------------------------------------------------------------------
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Linq;
using System.Linq.Expressions;
using System.Reflection;

namespace CarWale.Rules;

public sealed record Listing(
    int Id,
    string Make,
    string Model,
    int Year,
    decimal Price,
    int KmDriven,
    string FuelType,
    string City,
    bool IsCertified,
    int OwnerCount);

/// <summary>
/// Turns a POCO into a case-insensitive property bag once, using compiled
/// getters instead of reflection on every read.
/// </summary>
public static class RecordAdapter<T>
{
    private static readonly IReadOnlyDictionary<string, Func<T, object?>> Getters = Build();

    private static IReadOnlyDictionary<string, Func<T, object?>> Build()
    {
        var map = new Dictionary<string, Func<T, object?>>(StringComparer.OrdinalIgnoreCase);

        foreach (var prop in typeof(T).GetProperties(BindingFlags.Public | BindingFlags.Instance))
        {
            if (!prop.CanRead) continue;

            var param = Expression.Parameter(typeof(T), "x");
            var body = Expression.Convert(Expression.Property(param, prop), typeof(object));
            map[prop.Name] = Expression.Lambda<Func<T, object?>>(body, param).Compile();
        }

        return map;
    }

    public static IReadOnlyDictionary<string, object?> ToBag(T instance) =>
        Getters.ToDictionary(
            kv => kv.Key,
            kv => kv.Value(instance),
            StringComparer.OrdinalIgnoreCase);
}

public sealed class RuleEngine
{
    private readonly ConcurrentDictionary<string, IExpr> _cache = new(StringComparer.Ordinal);

    public IExpr Compile(string rule) => _cache.GetOrAdd(rule, Parser.Parse);

    public bool Matches(string rule, Listing listing)
    {
        var expr = Compile(rule);
        var ctx = new EvalContext
        {
            Record = RecordAdapter<Listing>.ToBag(listing),
            StrictFields = true
        };
        return Coerce.Truthy(expr.Evaluate(ctx));
    }

    public IReadOnlyList<Listing> Filter(string rule, IEnumerable<Listing> listings)
    {
        var expr = Compile(rule);
        return listings
            .Where(l => Coerce.Truthy(expr.Evaluate(new EvalContext
            {
                Record = RecordAdapter<Listing>.ToBag(l),
                StrictFields = true
            })))
            .ToList();
    }
}
```

```csharp
// ---------------------------------------------------------------------------
// Program.cs
// ---------------------------------------------------------------------------
using System;
using System.Collections.Generic;
using System.Linq;
using CarWale.Rules;

var listings = new List<Listing>
{
    new(1, "Maruti",  "Swift",  2021,  620000m, 18000, "Petrol",   "Pune",      true,  1),
    new(2, "Hyundai", "i20",    2019,  480000m, 42000, "Petrol",   "Mumbai",    false, 2),
    new(3, "Maruti",  "Baleno", 2018,  450000m, 55000, "Diesel",   "Pune",      true,  1),
    new(4, "Tata",    "Nexon",  2022, 1050000m,  9000, "Electric", "Bengaluru", true,  1),
    new(5, "Honda",   "City",   2020,  890000m, 31000, "Petrol",   "Mumbai",    false, 1),
};

var engine = new RuleEngine();

string[] rules =
{
    "price < 500000 AND (make = 'Maruti' OR year >= 2020)",
    "make IN ['Maruti', 'Tata'] AND isCertified",
    "year >= 2020 AND kmDriven < 20000",
    "NOT (fuelType = 'Diesel') AND price / 100000 < 7",
    "city CONTAINS 'mum' OR ownerCount = 1 AND price <= 700000",
};

foreach (var rule in rules)
{
    try
    {
        var ast = engine.Compile(rule);
        var matched = engine.Filter(rule, listings);

        Console.WriteLine();
        Console.WriteLine($"Rule: {rule}");
        Console.WriteLine($"  AST: {ast}");
        Console.WriteLine($"  Matched: " +
            (matched.Count == 0
                ? "(none)"
                : string.Join(", ", matched.Select(l => $"{l.Make} {l.Model}"))));
    }
    catch (Exception ex) when (ex is ParseException or TokenizeException or EvaluationException)
    {
        Console.WriteLine($"Rule failed: {rule}\n  {ex.Message}");
    }
}
```

Note the property-name matching: we built the bag with
`StringComparer.OrdinalIgnoreCase`, so the C# property `Price` resolves from the rule
text `price`. Business users should not have to learn PascalCase.

Also note the `catch` filter. In production, a bad rule must never take down the
consumer processing the RabbitMQ queue — you log it, mark the rule as broken, and
carry on with the rest.

---

## ✅ Applicability

Use Interpreter when **all** of these are true:

1. **You have an actual grammar.** Not "a few config options" — a thing with nesting,
   precedence, and composition. If a `field / op / value` table covers it, you don't
   need this.
2. **The grammar is small and stable.** Ten to twenty rules. If it's going to grow into
   a real programming language, stop and use a parser generator.
3. **Efficiency is not the top concern.** A tree walk allocates and does virtual
   dispatch per node. Fine for thousands of evaluations; not fine for a hot loop over
   millions.
4. **You need the sentences as data.** You can serialise the tree, show it in a UI,
   optimise it, translate it to SQL, or explain *why* a listing matched. A compiled
   lambda can't do any of that.

Point 4 is the one that actually keeps Interpreter alive. In our marketplace, the
product team wants a rule builder UI that renders the tree as nested condition cards,
and support wants "why did this listing get promoted?" traces. Both need the AST as a
first-class object. A `Func<Listing, bool>` gives you neither.

---

## ⚖️ Pros and cons

### Pros ✅

- **Easy to change and extend the grammar.** Adding `BETWEEN` means one new class plus
  a couple of lines in the parser. Nothing else changes.
- **Easy to implement the grammar.** Each rule is one small class with one small
  method. The recursion handles the combinatorics.
- **The AST is inspectable data.** Serialise it, diff it, render it, optimise it,
  explain it. This is the killer feature.
- **You control the surface area exactly.** Unlike `eval()`, a rule can only touch the
  fields you put in the context. No filesystem, no network, no prototype pollution.
- **Predictable failure.** Unknown field? You throw a specific error with a position.
  Try getting that out of a regex-based config hack.

### Cons ❌

- **Class explosion.** A grammar with many rules means many classes. Ours has ~8 and
  is already ~200 lines of expression code for a language with 10 operators. A real
  language would have dozens.
- **Slow.** Tree-walking is the slowest way to evaluate anything. Every node is a
  virtual call and usually a heap allocation for the boxed result.
- **You are writing a parser.** Error messages, precedence, associativity, escaping,
  Unicode, operator ambiguity — all your problem now. This is a well-understood but
  genuinely fiddly body of work, and the backtracking hack in our `parsePrimary` is a
  sign of how fast it gets awkward.
- **Two places to change for one feature.** Adding an operator touches the tokenizer,
  the parser, *and* the expression classes. Easy to get out of sync.
- **Hard to test exhaustively.** The input space is "all strings". You need a real
  test suite, ideally with property-based testing.

---

## 🔗 Relations with other patterns

### Interpreter IS a Composite

This is the most important structural fact about the pattern, and it's why the two
chapters belong together. Look at the definition of [Composite](../02-structural/03-composite.md):

> Compose objects into tree structures, then treat individual objects and compositions
> of objects uniformly.

Now look at our AST:

- `LiteralExpr` and `FieldExpr` are **leaves**.
- `AndExpr`, `OrExpr`, `ComparisonExpr` are **composites** holding child `Expr`s.
- Both implement the same `Expr` interface, so `AndExpr` cannot tell whether its left
  child is a leaf or a 50-node subtree.

That's Composite, exactly. Interpreter is Composite with one extra constraint: **the
tree's shape mirrors a grammar, and the uniform operation is "evaluate me".**

```
   Composite                          Interpreter
   ─────────                          ───────────
   Component                          AbstractExpression
     Leaf                               TerminalExpression
     Composite ──has──▶ Component       NonterminalExpression ──has──▶ AbstractExpression

   Operation: whatever the domain      Operation: interpret(context)
              needs (render, price,
              getSize…)                Tree shape: dictated by a grammar
```

So: every Interpreter is a Composite. Not every Composite is an Interpreter (a UI
widget tree is a Composite but has no grammar).

### Visitor — how you do *more* than evaluate

Our expression classes each have `evaluate()`. But in a real system you want more than
evaluation. You want:

- **Pretty-print** the rule back to canonical text (for the rule-editor UI).
- **Translate to SQL** so the database can do the filtering instead of your app.
- **Optimise** the tree — constant-fold `2 + 3` into `5`, drop `x AND TRUE`.
- **Validate** — collect every field name used so you can check them against a schema.
- **Explain** — produce a trace of which subexpressions were true and why.

If you add all of those as methods on every expression class, each class balloons and
every new operation forces you to edit all eight classes. That's the *Open/Closed*
violation [Visitor](../03-behavioral/10-visitor.md) exists to solve.

The Visitor version: keep the node classes as dumb data, move each operation into its
own visitor class.

```ts
// ---------------------------------------------------------------------------
// visitor.ts — one operation per class, instead of one method per node class
// ---------------------------------------------------------------------------

export interface ExprVisitor<R> {
  visitLiteral(e: LiteralExpr): R;
  visitField(e: FieldExpr): R;
  visitList(e: ListExpr): R;
  visitAnd(e: AndExpr): R;
  visitOr(e: OrExpr): R;
  visitNot(e: NotExpr): R;
  visitComparison(e: ComparisonExpr): R;
  visitArithmetic(e: ArithmeticExpr): R;
  visitNegate(e: NegateExpr): R;
}

// Each node gains exactly one method: accept().
export interface VisitableExpr extends Expr {
  accept<R>(visitor: ExprVisitor<R>): R;
}

// e.g. on AndExpr:
//   accept<R>(v: ExprVisitor<R>): R { return v.visitAnd(this); }

/**
 * Operation 1: turn a rule into a parameterised SQL WHERE clause,
 * so the database does the filtering.
 *
 * Parameterised — never string-interpolate a literal into SQL.
 */
export class SqlVisitor implements ExprVisitor<string> {
  readonly parameters: unknown[] = [];

  private param(value: unknown): string {
    this.parameters.push(value);
    return `@p${this.parameters.length - 1}`;
  }

  visitLiteral(e: LiteralExpr): string   { return this.param(e.value); }
  visitField(e: FieldExpr): string       { return quoteIdentifier(e.name); }
  visitList(e: ListExpr): string {
    return `(${e.items.map((i) => (i as VisitableExpr).accept(this)).join(', ')})`;
  }

  visitAnd(e: AndExpr): string {
    return `(${(e.left as VisitableExpr).accept(this)} AND ${(e.right as VisitableExpr).accept(this)})`;
  }
  visitOr(e: OrExpr): string {
    return `(${(e.left as VisitableExpr).accept(this)} OR ${(e.right as VisitableExpr).accept(this)})`;
  }
  visitNot(e: NotExpr): string {
    return `NOT (${(e.operand as VisitableExpr).accept(this)})`;
  }

  visitComparison(e: ComparisonExpr): string {
    const left = (e.left as VisitableExpr).accept(this);
    const right = (e.right as VisitableExpr).accept(this);
    switch (e.op) {
      case '=':  return `${left} = ${right}`;
      case '!=': return `${left} <> ${right}`;
      case 'IN': return `${left} IN ${right}`;
      case 'CONTAINS': return `${left} LIKE '%' + ${right} + '%'`;
      default:   return `${left} ${e.op} ${right}`;
    }
  }

  visitArithmetic(e: ArithmeticExpr): string {
    return `(${(e.left as VisitableExpr).accept(this)} ${e.op} ${(e.right as VisitableExpr).accept(this)})`;
  }
  visitNegate(e: NegateExpr): string {
    return `-(${(e.operand as VisitableExpr).accept(this)})`;
  }
}

/** Operation 2: collect every field a rule references, to validate against a schema. */
export class FieldCollector implements ExprVisitor<void> {
  readonly fields = new Set<string>();

  visitLiteral(): void {}
  visitField(e: FieldExpr): void { this.fields.add(e.name); }
  visitList(e: ListExpr): void { e.items.forEach((i) => (i as VisitableExpr).accept(this)); }
  visitAnd(e: AndExpr): void { (e.left as VisitableExpr).accept(this); (e.right as VisitableExpr).accept(this); }
  visitOr(e: OrExpr): void   { (e.left as VisitableExpr).accept(this); (e.right as VisitableExpr).accept(this); }
  visitNot(e: NotExpr): void { (e.operand as VisitableExpr).accept(this); }
  visitComparison(e: ComparisonExpr): void { (e.left as VisitableExpr).accept(this); (e.right as VisitableExpr).accept(this); }
  visitArithmetic(e: ArithmeticExpr): void { (e.left as VisitableExpr).accept(this); (e.right as VisitableExpr).accept(this); }
  visitNegate(e: NegateExpr): void { (e.operand as VisitableExpr).accept(this); }
}

function quoteIdentifier(name: string): string {
  if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(name)) {
    throw new Error(`Refusing to emit unsafe identifier '${name}'`);
  }
  return `[${name}]`;
}
```

Now `SqlVisitor` lets you push the same business rule down into SQL Server:

```ts
const ast = Parser.parse(`make IN ['Maruti','Tata'] AND price < 500000`);
const sql = new SqlVisitor();
const where = (ast as VisitableExpr).accept(sql);
// where      => "([make] IN (@p0, @p1) AND ([price] < @p2))"
// sql.parameters => ['Maruti', 'Tata', 500000]
```

**This is the real payoff of Interpreter.** One rule string, written once by a business
user, evaluated three different ways: in-memory against a RabbitMQ message, translated
to a SQL WHERE clause for the listings table, and pretty-printed back into the admin UI.
None of those are possible if you compiled straight to a lambda.

The trade-off is the classic one from the [Visitor](../03-behavioral/10-visitor.md)
chapter: adding a new *operation* is now cheap (one new visitor), but adding a new
*node type* is expensive (every visitor must handle it). For a DSL that's exactly the
right trade — grammars stabilise, but the number of things you want to do with a parsed
rule keeps growing.

### Other relations

| Pattern | Relationship |
|---|---|
| [Composite](../02-structural/03-composite.md) | Interpreter's AST *is* a Composite. See above. |
| [Visitor](../03-behavioral/10-visitor.md) | The standard way to add operations over the AST without touching node classes. |
| [Iterator](../03-behavioral/03-iterator.md) | Used to traverse the tree — a visitor is usually a hand-rolled traversal, but a generic AST walker is an Iterator. |
| [Flyweight](../02-structural/06-flyweight.md) | GoF explicitly suggests sharing terminal nodes. A rule with `fuelType = 'Petrol'` mentioned 40 times can share one `LiteralExpr('Petrol')`. Worth it only for huge ASTs. |
| [Builder](../01-creational/03-builder.md) | A fluent API for constructing the AST in code — `Rule.Where("price").LessThan(500000).And(...)` — instead of parsing a string. Very common as a type-safe alternative to the DSL. |
| [Strategy](../03-behavioral/08-strategy.md) | If your grammar is really just "pick one of N behaviours", you wanted Strategy, not Interpreter. Check this before you start. |
| [Factory Method](../01-creational/01-factory-method.md) | The parser is effectively a big factory: each grammar method decides which expression class to instantiate. |
| [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) | An alternative shape for simple linear rule sets — handlers in sequence, first match wins. Much simpler than a grammar when there's no nesting. |

---

## 🚫 When NOT to use it (almost always)

Let's be blunt. Interpreter is the pattern most likely to be a mistake when you reach
for it. Here is the honest decision tree:

```
Do users need to write logic you can't hard-code?
│
├── NO ──▶ Just write the code. Use Strategy or a simple if. Stop here.
│
└── YES
    │
    ├── Is the "language" really just N fixed choices?
    │   └── YES ──▶ Enum + Strategy. Stop here.
    │
    ├── Is it flat key/op/value with no nesting?
    │   └── YES ──▶ A config table + a small evaluator loop. Stop here.
    │
    ├── Does a battle-tested rules engine already do this?
    │   └── YES ──▶ Use it. (See alternatives below.) Stop here.
    │
    ├── Is it a real, general-purpose language?
    │   └── YES ──▶ Parser generator (ANTLR) or an existing embedded language.
    │               Do NOT hand-roll. Stop here.
    │
    └── Small, stable grammar + you need the AST as inspectable data?
        └── YES ──▶ OK. Interpreter is genuinely the right call.
```

Concrete anti-cases:

**"I need users to write arbitrary expressions."** *Arbitrary* is the red flag. Once the
grammar is unbounded you are writing a language implementation, which is a multi-year
project, not a sprint. Either narrow the grammar hard or embed an existing sandboxed
language.

**"It'll only ever be a few operators."** Every DSL says this. Then someone asks for
date arithmetic, then string functions, then a `CASE WHEN`, then "can I call another
rule from inside a rule?", and now you own a badly-specified Turing-complete language
with no debugger, no formatter, and no error recovery. Budget for that before you start.

**"Performance doesn't matter."** It does the moment someone runs the rule over the
whole listings table. Tree-walking `Evaluate` on 2 million rows with boxed `object?`
returns will be slower than you expect by roughly an order of magnitude. If you know
the shape in advance, compile instead of interpret.

**"It's just for internal admins."** Internal users paste things from Slack. A rule that
throws inside a RabbitMQ consumer and isn't caught will kill your queue processing at
3am. Interpreting untrusted input is a production concern regardless of who typed it.

---

## 🛠️ Modern alternatives

### C# — `System.Linq.Expressions` (compile, don't interpret)

This is the single biggest reason you rarely hand-roll Interpreter in .NET. The BCL
ships an expression-tree library, and you can *compile it to IL at runtime*. Same AST
idea, but you get native-speed execution and the tree is still inspectable.

```csharp
// ---------------------------------------------------------------------------
// LinqExpressionCompiler.cs — same AST, but compiled to a real delegate
// ---------------------------------------------------------------------------
using System;
using System.Linq.Expressions;
using System.Reflection;

namespace CarWale.Rules;

public static class LinqExpressionCompiler
{
    /// <summary>
    /// Walks OUR ast and emits a System.Linq.Expressions tree, then compiles it.
    /// The result runs at roughly the speed of hand-written C#.
    /// </summary>
    public static Func<Listing, bool> Compile(IExpr ast)
    {
        var param = Expression.Parameter(typeof(Listing), "listing");
        var body = Build(ast, param);

        // Coerce whatever we produced into bool.
        if (body.Type != typeof(bool))
            body = Expression.Convert(body, typeof(bool));

        return Expression.Lambda<Func<Listing, bool>>(body, param).Compile();
    }

    private static Expression Build(IExpr node, ParameterExpression param) => node switch
    {
        LiteralExpr lit => Expression.Constant(lit.Value, lit.Value?.GetType() ?? typeof(object)),

        FieldExpr f => BuildFieldAccess(f.Name, param),

        NotExpr n => Expression.Not(AsBool(Build(n.Operand, param))),

        AndExpr a => Expression.AndAlso(
            AsBool(Build(a.Left, param)),
            AsBool(Build(a.Right, param))),

        OrExpr o => Expression.OrElse(
            AsBool(Build(o.Left, param)),
            AsBool(Build(o.Right, param))),

        ComparisonExpr c => BuildComparison(c, param),

        ArithmeticExpr ar => BuildArithmetic(ar, param),

        NegateExpr neg => Expression.Negate(AsDouble(Build(neg.Operand, param))),

        _ => throw new NotSupportedException($"Cannot compile node type {node.GetType().Name}")
    };

    private static Expression BuildFieldAccess(string name, ParameterExpression param)
    {
        var prop = typeof(Listing).GetProperty(
            name,
            BindingFlags.Public | BindingFlags.Instance | BindingFlags.IgnoreCase);

        if (prop is null)
            throw new EvaluationException($"Unknown field '{name}' on {typeof(Listing).Name}");

        return Expression.Property(param, prop);
    }

    private static Expression BuildComparison(ComparisonExpr c, ParameterExpression param)
    {
        var left = Build(c.Left, param);
        var right = Build(c.Right, param);

        // Promote both sides to a common numeric type where relevant.
        if (IsNumeric(left.Type) && IsNumeric(right.Type))
        {
            left = Expression.Convert(left, typeof(double));
            right = Expression.Convert(right, typeof(double));
        }

        return c.Op switch
        {
            "="  => BuildEquality(left, right, negate: false),
            "!=" => BuildEquality(left, right, negate: true),
            "<"  => Expression.LessThan(left, right),
            "<=" => Expression.LessThanOrEqual(left, right),
            ">"  => Expression.GreaterThan(left, right),
            ">=" => Expression.GreaterThanOrEqual(left, right),
            _    => throw new NotSupportedException(
                        $"Operator '{c.Op}' is not supported by the compiler path")
        };
    }

    private static Expression BuildEquality(Expression left, Expression right, bool negate)
    {
        if (left.Type == typeof(string) && right.Type == typeof(string))
        {
            var equalsMethod = typeof(string).GetMethod(
                nameof(string.Equals),
                new[] { typeof(string), typeof(string), typeof(StringComparison) })!;

            var call = Expression.Call(
                equalsMethod,
                left,
                right,
                Expression.Constant(StringComparison.OrdinalIgnoreCase));

            return negate ? Expression.Not(call) : call;
        }

        return negate ? Expression.NotEqual(left, right) : Expression.Equal(left, right);
    }

    private static Expression BuildArithmetic(ArithmeticExpr a, ParameterExpression param)
    {
        var left = AsDouble(Build(a.Left, param));
        var right = AsDouble(Build(a.Right, param));

        return a.Op switch
        {
            '+' => Expression.Add(left, right),
            '-' => Expression.Subtract(left, right),
            '*' => Expression.Multiply(left, right),
            '/' => Expression.Divide(left, right),
            _   => throw new NotSupportedException($"Arithmetic operator '{a.Op}'")
        };
    }

    private static Expression AsBool(Expression e) =>
        e.Type == typeof(bool) ? e : Expression.Convert(e, typeof(bool));

    private static Expression AsDouble(Expression e) =>
        e.Type == typeof(double) ? e : Expression.Convert(e, typeof(double));

    private static bool IsNumeric(Type t) =>
        t == typeof(int) || t == typeof(long) || t == typeof(double) ||
        t == typeof(decimal) || t == typeof(float) || t == typeof(short);
}
```

Usage:

```csharp
var ast = Parser.Parse("price < 500000 AND (make = 'Maruti' OR year >= 2020)");

// Compile once per rule, at startup or on first use.
Func<Listing, bool> predicate = LinqExpressionCompiler.Compile(ast);

// Then run it millions of times at near-native speed.
var matched = listings.Where(predicate).ToList();
```

**This is usually the right architecture in C#:** keep the Interpreter AST as your
*canonical, inspectable* representation, then compile it to a delegate for execution.
You get the best of both — the tree for the admin UI and the SQL translation, and the
compiled lambda for throughput.

And because `System.Linq.Expressions` is what LINQ providers consume, building an
`Expression<Func<Listing, bool>>` instead of `Func<Listing, bool>` lets Entity Framework
translate your rule into a real SQL query. That's the same SqlVisitor idea, done by a
library that has already solved all the edge cases.

```csharp
// Expression<Func<...>> instead of Func<...> — EF Core can translate this.
public static Expression<Func<Listing, bool>> CompileForEf(IExpr ast)
{
    var param = Expression.Parameter(typeof(Listing), "listing");
    var body = Build(ast, param);
    return Expression.Lambda<Func<Listing, bool>>(AsBool(body), param);
}

// dbContext.Listings.Where(CompileForEf(ast)).ToListAsync();
//   ──▶ becomes a SQL WHERE clause executed by the database
```

### C# — Roslyn

If you want the user to write actual C#, the Roslyn compiler platform can compile and
run C# at runtime. It is the nuclear option: full language, full power, full risk.
It needs real sandboxing and a separate process, and start-up cost is significant.
Reach for it when you genuinely need a programming language, not a rule DSL.

### ANTLR

When your grammar is big enough that hand-rolling hurts, **ANTLR** generates the
tokenizer and parser from a grammar file, for C#, Java, TypeScript, C++ and more. You
write the grammar once and get a parser plus a visitor/listener base class for free.

```antlr
// CarRule.g4 — sketch of the same grammar as a generated parser would see it
grammar CarRule;

expression : orExpr ;
orExpr     : andExpr ( OR andExpr )* ;
andExpr    : notExpr ( AND notExpr )* ;
notExpr    : NOT notExpr | primary ;
primary    : LPAREN expression RPAREN | comparison ;
comparison : arith ( compareOp arith )? ;
arith      : arith ( MUL | DIV ) arith
           | arith ( ADD | SUB ) arith
           | atom ;
atom       : NUMBER | STRING | BOOLEAN | IDENTIFIER | LPAREN arith RPAREN ;
compareOp  : EQ | NEQ | LT | LTE | GT | GTE ;

AND : 'AND' ; OR : 'OR' ; NOT : 'NOT' ;
// ... lexer rules ...
```

The generated visitor is the *same pattern* as our `ExprVisitor`. ANTLR just writes the
boring parts for you, and handles error recovery far better than a hand-rolled parser.

Trade-off: a build step, a runtime dependency, and a grammar file that is its own skill
to learn. Worth it past maybe 20 grammar rules.

### TypeScript — jsep

`jsep` (JavaScript Expression Parser) parses a JavaScript-*expression* subset — no
statements, no function declarations — into a standard ESTree-shaped AST. You then walk
that AST yourself. It is a good fit when your DSL is basically "JS expressions over a
data object", because you skip the tokenizer and parser entirely and keep only the
evaluator, which is the part with your business logic in it.

```ts
// Sketch. Check the current jsep docs for the exact API before you rely on this.
import jsep from 'jsep';

const ast = jsep("price < 500000 && (make === 'Maruti' || year >= 2020)");

// ast is an ESTree-ish node tree:
//   { type: 'LogicalExpression', operator: '&&', left: {...}, right: {...} }
//
// You write ONLY the evaluator — the part that decides what `price` resolves to
// and which operators you allow. That's still Interpreter, minus the parser.

function evaluate(node: any, ctx: Record<string, unknown>): unknown {
  switch (node.type) {
    case 'Literal':    return node.value;
    case 'Identifier': return ctx[node.name];
    case 'LogicalExpression':
      return node.operator === '&&'
        ? evaluate(node.left, ctx) && evaluate(node.right, ctx)
        : evaluate(node.left, ctx) || evaluate(node.right, ctx);
    case 'BinaryExpression': {
      const l = evaluate(node.left, ctx) as never;
      const r = evaluate(node.right, ctx) as never;
      switch (node.operator) {
        case '<':   return l < r;
        case '<=':  return l <= r;
        case '>':   return l > r;
        case '>=':  return l >= r;
        case '===': case '==': return l === r;
        case '!==': case '!=': return l !== r;
        case '+':   return (l as number) + (r as number);
        case '-':   return (l as number) - (r as number);
        case '*':   return (l as number) * (r as number);
        case '/':   return (l as number) / (r as number);
        default: throw new Error(`Operator ${node.operator} not allowed`);
      }
    }
    case 'UnaryExpression':
      if (node.operator === '!') return !evaluate(node.argument, ctx);
      if (node.operator === '-') return -(evaluate(node.argument, ctx) as number);
      throw new Error(`Unary ${node.operator} not allowed`);
    default:
      throw new Error(`Node type ${node.type} not allowed`);
  }
}
```

The `default: throw` is the security boundary. Whitelist node types explicitly; never
fall through to "evaluate whatever this is".

### TypeScript — a Pratt parser

If you *do* hand-roll, a **Pratt parser** (top-down operator precedence) is usually a
better shape than recursive descent for expression languages. Instead of one method per
precedence level, you register a *binding power* per operator and one generic loop
handles all of them. Adding an operator becomes a one-line table entry, and the
backtracking hack from our `parsePrimary` disappears.

```ts
// Sketch of the core loop — the idea, not a full implementation.
const BINDING_POWER: Record<string, number> = {
  'OR': 10, 'AND': 20,
  '=': 30, '!=': 30, '<': 30, '<=': 30, '>': 30, '>=': 30,
  '+': 40, '-': 40,
  '*': 50, '/': 50,
};

function parseExpr(p: TokenStream, minBp = 0): Expr {
  let left = parsePrefix(p);            // literal, field, '(' expr ')', unary

  for (;;) {
    const op = p.peekOperator();
    if (!op) break;

    const bp = BINDING_POWER[op];
    if (bp === undefined || bp < minBp) break;

    p.advance();
    const right = parseExpr(p, bp + 1);  // +1 => left-associative
    left = makeBinary(left, op, right);
  }

  return left;
}
```

That one loop replaces `parseOr`, `parseAnd`, `parseArith`, and `parseTerm` from our
recursive-descent parser. For a grammar that will grow, write it this way from day one.

### Existing rules engines

Before writing any of this: check whether something already does it. The category exists
and is mature — JSON-rules engines in the Node ecosystem, JSON Logic style evaluators
(a JSON AST you build in a UI, no parser needed), and full business-rules platforms on
the .NET and JVM sides. Evaluate one properly before you commit to owning a parser.

The JSON-AST approach in particular is worth a hard look for a marketplace: if your
rules are built in a drag-and-drop UI, **you never need a text parser at all.** The UI
produces the tree directly, you store it as JSON, and you only write the evaluator —
which is the Interpreter's expression classes and nothing else. That skips the hardest
and most bug-prone half of the work.

```json
{
  "op": "and",
  "args": [
    { "op": "<",  "args": [{ "field": "price" }, { "const": 500000 }] },
    { "op": "or", "args": [
      { "op": "=",  "args": [{ "field": "make" }, { "const": "Maruti" }] },
      { "op": ">=", "args": [{ "field": "year" }, { "const": 2020 }] }
    ]}
  ]
}
```

That JSON deserialises straight into our expression classes. No tokenizer, no parser,
no ambiguity, no error messages to write. Strongly recommended as the *default* choice
unless a text DSL is genuinely a product requirement.

---

## 🌍 Where Interpreter genuinely shows up

The pattern is not dead, it's just specialised. Here's where you actually meet it:

### Regex engines

A regular expression is a sentence in a tiny language. `^a(b|c)*d$` is parsed into a
tree of nodes — concatenation, alternation, repetition, character class, anchor — and
then either walked directly (a backtracking engine) or compiled into a state machine
(an NFA/DFA engine). The parsed-tree stage is Interpreter, textbook.

This is also a good illustration of the pattern's ceiling: every serious regex engine
*compiles* the tree rather than walking it, because walking is too slow for the volume.
Same story as `System.Linq.Expressions` above.

### SQL query planners

Your database does exactly what we did, at industrial scale:

```
SQL text
   │ tokenizer
   ▼
tokens
   │ parser
   ▼
parse tree  ──▶ semantic analysis (do these tables/columns exist?)
   │
   ▼
logical plan (a tree: Filter over Join over Scan)
   │ optimiser — a set of Visitors rewriting the tree
   ▼
physical plan (Hash Join vs Nested Loop, which index)
   │
   ▼
execution — iterators pulling rows through the tree
```

The logical plan is an expression tree of relational operators. The optimiser is a pile
of Visitors that rewrite it (push predicates down, reorder joins, eliminate subqueries).
The executor walks it. Interpreter plus Visitor plus Iterator, all three.

And the `WHERE` clause specifically — that's our rule engine, only better. Which is why
the SqlVisitor above is often the smartest move: don't evaluate the rule yourself,
translate it and let the database's far more sophisticated interpreter do the work.

### Spreadsheet formula engines

`=IF(B2>100, B2*0.9, B2)` in a cell. Parsed into a tree, evaluated against a context
(the sheet), with a dependency graph so that changing `B2` recomputes only what depends
on it. The formula bar is a DSL editor; the cell is the context; the recalculation
engine is the interpreter. Millions of non-programmers use one every day.

### Feature-flag and targeting rule evaluators

Every feature-flag platform lets you write targeting rules — "enable for users where
`country IN ['IN','LK'] AND plan = 'pro' AND signupDate > '2024-01-01'`". That rule is
distributed to SDKs as a JSON AST and evaluated client-side against the user context.

This is *exactly* our rule engine, and it's the strongest argument for the JSON-AST
approach: the rule is authored in a web UI, serialised as a tree, shipped over the wire,
and evaluated by a tiny interpreter embedded in every SDK. No parser anywhere in the
pipeline.

### Search query DSLs

The search box on your own marketplace. When a user types
`maruti swift -diesel city:pune price:<500000`, something has to turn that into a
structured query. Lucene-family engines define a query DSL that parses into a tree of
query clauses — boolean must/should/must_not, range, term, phrase — which the engine
then executes.

Two layers of Interpreter, in fact: the human-typed query string is parsed into a query
AST, and the JSON query DSL is *also* an AST, just pre-parsed.

### Others worth knowing

- **Template engines** — Razor, Handlebars, Liquid. Template text parses into a node
  tree (text, interpolation, loop, conditional) which is then either walked or compiled.
- **Build tools and config languages** — anything with conditionals in YAML/HCL/etc. is
  running a small interpreter over an expression language.
- **Protocol buffers / GraphQL** — a `.proto` file or a GraphQL query is parsed into an
  AST, then compiled or executed. GraphQL's resolver walk over the query AST is
  Interpreter with the schema as context.
- **Pricing and promotion rules** — "10% off if cart total > ₹5000 and user is a repeat
  buyer" — the same rule-engine shape as our listings example.

---

## 🧾 Recap

**What it is:** one class per grammar rule, a shared `interpret(context)` method, and a
tree of those objects representing a sentence. Evaluate by walking the tree.

**The parts:** `AbstractExpression` (the interface), `TerminalExpression` (leaves:
literals, field reads), `NonterminalExpression` (branches: AND, OR, comparison),
`Context` (what the leaves resolve against), `Client` (builds the tree, calls
`interpret`).

**Its true nature:** Interpreter is [Composite](../02-structural/03-composite.md) with a
grammar. Add [Visitor](../03-behavioral/10-visitor.md) the moment you want more than one
operation over the tree, which is almost immediately.

**When to use it:** small, stable grammar; performance not critical; and — the real
reason — you need the parsed rule as **inspectable data** so you can render it, explain
it, optimise it, or translate it to SQL.

**When not to:** almost always. If it's N fixed choices, use
[Strategy](../03-behavioral/08-strategy.md). If it's flat config, use a table. If it's a
real language, use ANTLR or an embedded language. If it's C# and you need speed, keep
the AST but compile to `System.Linq.Expressions`. If the rules are authored in a UI,
serialise a JSON AST and skip the parser entirely — that's the highest-value shortcut
in this whole chapter.

**Where it lives for real:** regex engines, SQL planners, spreadsheet formulas,
feature-flag targeting, search query DSLs, template engines. Not in most application
code — and that's fine. Knowing the pattern means you can read those systems, and it
means that on the rare day you *do* need a small DSL, you'll build it deliberately
instead of growing an `eval()` call into a liability.

---

### 🔎 Go deeper

- The structural twin: [Composite](../02-structural/03-composite.md)
- Adding operations over the tree: [Visitor](../03-behavioral/10-visitor.md)
- The simpler alternative you should rule out first: [Strategy](../03-behavioral/08-strategy.md)
- For linear rule chains without nesting: [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md)
- Type-safe AST construction in code: [Builder](../01-creational/03-builder.md)
- Sharing repeated terminal nodes: [Flyweight](../02-structural/06-flyweight.md)
- Design principles behind the Visitor trade-off: [SOLID and principles](./01-solid-and-principles.md)
