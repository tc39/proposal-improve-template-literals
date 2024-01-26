# Raw string literals

## TLDR.

```javascript
let query = `
    select * 
    from \`users\` 
    where \`name\` = ?
`
```

Escaping is annoying, and `String.raw` or `String.dedent` doesn't help in this case. We need syntax to solve the issue:

````c#
// syntax TBD, just use @sken130 's strawperson draft as demo
let query = @``
    select * 
    from `users` 
    where `name` = ?
    ``
````

## Status

- Stage: 0
- Champion: HE Shi-Jun (hax)
- Authors: @hax, @sken130

## Motivation

JavaScript lacks a general way to create simple string literals that can contain effectively any arbitrary text. Using template literals with `String.raw` built-in tag function could avoid most escaping, but unfortunately can't include `` ` `` and `${` verbatimly as they are delimiters of template literals. This prevents easily having literals containing other programming languages (notably, JavaScript itself) and text formats (for example, Markdown) in them.

All current approaches to form these literals in JavaScript today always force the user to manually escape the contents, or use some other hacks. Editing at that point can be highly annoying as the escaping cannot be avoided and must be dealt with whenever it arises in the contents. This is particularly painful for the text contains both backslashes and backticks, or the multiple lines text contains backticks.

The crux of the problem is that all our strings have a fixed start/end delimiter. As long as that is the case, we will always have to use an escaping or substitution mechanism as the string contents may need to specify that end delimiter in their contents. This is particularly problematic as that delimiter `` ` `` is common in many languages.

To address this, this proposal allows for flexible start and end delimiters so that they can always be made in a way that will not conflict with the content of the string.

## Core Goals

1. Provide a mechanism that will allow *all* string values to be provided by the user without the need for *any* escape-sequences or substitutions whatsoever. Because all strings must be representable without escape-sequences or substitutions, it must always be possible for the user to specify delimiters that will be guaranteed to not collide with any text contents.
2. Support interpolations in the same fashion. As above, because *all* strings must be representable without escapes or substitutions, it must always be possible for the user to specify an interpolation delimiter that will be guaranteed to not collide with any text contents.
3. Support tag functions as current tagged template literals do.

## Possible Extra Goals
4. Multiline string literals should look pleasant in code and should not make indentation within the compilation unit look strange. Importantly, literal values that themselves have no indentation should not be forced to occupy the first column of the file as that can break up the flow of code and will look unaligned with the rest of the code that surrounds it. This behavior should be easy to override while keeping literals clear and easy to read.
    - Note, this goal might also be meet with `String.dedent` proposal, but syntax solution likely has better ergonomics and other benefits.
5. Allow mechnism like [Markdown info string](https://spec.commonmark.org/0.30/#info-string), which can be leavaged by tools (for example, use for syntax highlighting).

## Possible Solution

There are many possible syntax options, here document style syntax, Swift/Rust style (```#`raw string`#```) syntax, Markdown style syntax, etc. The champion plan to investigate different syntax options when the proposal approved as stage 1.

Currently let's just use @sken130 's strawperson draft which heavily inspired by C# 11 raw string literal syntax.

### 1\. The string sequence should start with an `@` and at least (can be more than) 2 backtick characters, and close with an equal number of backtick characters without `@`.

````````````````js
const str = @``
I am a string
``
````````````````

### 2\. These patterns don't need to be escaped

Obviously, characters such as @, double quote, single quote, and backslash won't need to be escaped

````````````````js
const mysqlQuery = @``
    SELECT * FROM `Strange table name` where `strange column name` = 'abcde';
    ``
````````````````

````````````````js
const str = @``
I would like to request "2 leave days" that start from '11/26/2022' and end on '11/27/2022'.
\_/
If any enquiry, please send email to ken@example.com
``
````````````````
The @, ", ', and \ characters will be stored in str as-is, as visibly shown in source code, no need \", \', \\.

1 backtick character doesn't need to be escaped either

````````````````js
const str = @``
I would like to request `2 leave days` that ......
``
````````````````
And the
```
@`
```
sequence doesn't need to be escaped either, since it is not equal to the closing delimiter.

### 3\. Here comes the problem: what if we want to embed 2 or more backtick characters without escaping? In such case, the raw string literal needs to start and end with more backtick characters:

Embedding 2 backtick characters would require opening with an @ and 3 backticks and closing with 3 backticks:
````````````````js
const javaScriptTutorial = @```
   const emptyString = ``  // Yay, backtick quotes can be an empty string too
   ```
````````````````
(the 2 backtick ``` characters will be represented as-is, without the need of escaping)

Embedding 3 backtick characters (a common example is embedding markdown in JavaScript) would require opening with an @ and 4 backticks and closing with 4 backticks:
````````````````js
const markdownExample = @````
    ```json
    {
      "firstName": "John",
      "lastName": "Smith",
      "age": 25
    }
    ```
    ````
````````````````
(the 3 backtick ``` characters will be represented as-is, without the need of escaping)

If we want to represent 4 backticks in the content without escaping, we need to delimit the raw string by 5 backtick characters, and so on.
In this design, we can put arbitrary number of backticks in the content as long as we delimit the raw string by opening and closing with more backtick characters.
Such syntax should not be clumsy, as such kind of strings are rare (we want to cover all corner cases though).

### 4\. Indentation - Any whitespace to the left of the closing delimiter (```, or ````, or `````, ...) will be removed from the string literal

````````````````js
  const markdownExample = @````
     |```json
     |{
     |  "firstName": "John",
     |  "lastName": "Smith",
     |  "age": 25
     |}
     |```
     |
      ````
````````````````
Note: The | characters are not really in the string, they're for illustrating how the string is indented and that the whitespaces at | and on the left of | are not captured in the string.

It is equivalent to
````````````````js
    const markdownExample = @````
```json
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```

````
````````````````

Any characters on the left of the closing delimiter will trigger compilation error:
````````````````js
    const markdownExample = @````
        ```json
        {
          "firstName": "John",
          "lastName": "Smith",
          "age": 25
        }
        ```
       a   // The character a is illegal here and should give compilation error rather than ignoring it.
    b   ````  // The character b is also illegal here, the closing delimiter must be on its own line. Should give compilation error rather than ignoring it.
````````````````

### 5\. Indentation - indentation whitespace must be consistent

If the closing delimiter has 8 spaces to the left, then all lines in the content must start with 8 spaces, not tab characters. They can have tab or space characters after the initial 8 spaces though.

If the closing delimiter has 2 tabs to the left, then all lines in the content must start with 2 tabs, not space characters. They can have space or tab characters after the initial 2 tabs though.

### 6\. Interpolation - You know it

````````````````js
let s = @``
  My name is ${myName}
  ``
````````````````

### 7\. What if I want to include ${myName} as part of the content, not interpolation?

The answer borrowed from C# 11 raw string literal is, to include more @ characters at the beginning delimiter.

The number of @ characters at the opening delimiter, will control how many $ characters are needed to start an interpolation:
````````````````js
let s1 = @@``
  My name is ${myName}
  ``    // No interpolation, ${myName} is now stored in the string as-is
let s1 = @@``
  My name is $${myName}
  ``    // Interpolation will happen
````````````````

````````````````js
const myName = "Ann"
console.log(@@``
  JavaScript tutorial: If you write console.log(`my name is ${myName}`), it will print "my name is $${myName}" in the results (not including "")
``)
````````````````

will print
```
JavaScript tutorial: If you write console.log(`my name is ${myName}`), it will print "my name is Ann" in the results (not including "")
```

More examples
````````````````js
@@@```
  My name is $${myName}
  ```    // No interpolation, ${myName} is now stored in the string as-is
@@@```
  My name is $$${myName}
  ```    // Interpolation will happen
````````````````

### 8\. Tagged string

````````````````js
let result = tag@``
  should also support tag function
  ``
````````````````

## 9\. More rules on multi-line raw string literal

- Both opening and closing quote characters must be on different lines.
- Whitespace following the opening quote on the same line is ignored.
- Any non-whitespace characters (except comments) following the opening quote on the same line are illegal, and will be treated as unterminated single-line raw string literal.
- Whitespace only lines below the opening quote are included in the string literal.

### 10. Why this syntax

#### Why not just open with ```

Because ``` is not fully backwards compatible. See https://github.com/tc39/proposal-string-dedent/issues/40, and https://github.com/tc39/proposal-string-dedent/issues/8, and https://gist.github.com/michaelficarra/70ce798feb25fdc91508f387190053a1, and my reply before this proposal https://es.discourse.group/t/raw-string-literals-that-can-contain-any-arbitrary-text-without-the-need-for-special-escape-sequences/1757/2

#### Why the @ character for opening delimiter

Because @``, @```, @````, ... are never valid JavaScript syntax before, so no backwards compatibility issue.

We also can't use $\`\`, $\`\`\`, ..., as $ is a valid variable identifier (jQuery).

#### Why not just use other delimiter character/sequence than backtick, double quotes or single quote?

- If the delimiter character/sequence is fixed, no matter what delimiting character/sequence you choose, you cannot embed the closing delimiter character/sequence in the content without escaping. One of the goals of this feature, is to be able to embed arbitrary text without escaping, and to do so, the opening and closing delimiter sequence have to be flexible.
- We may use other flexible sequence than @``, but not sure if this would waste the room of possible syntaxes for other future enhancements.

### 11. Use cases of raw string literal

#### (a) MySQL query

With MySQL, the following situations are common

````js
const searchUser = () => {
    const query = `select *
from \`users\`
where \`name\` = ?`
}
````
Allowing raw string representation without escaping, and with proper indentation, will make it less troublesome to maintain and less error prone.
````js
const searchUser = () => {
    const query = @``
        select *
        from `users`
        where `name` = ?
        ``
    ......
}
````

#### (b) Markdown embedding or generation

Without raw string literal feature, we have to do this (have to escape each backtick):
````````js
const generateMarkdown = () => {
    const markdownExample = `\`\`\`json
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
\`\`\`
`
    doSomething(markdownExample)  // Output/process the markdown somewhere
}
````````
or this (have to escape each double quote):
````````js
const generateMarkdown = () => {
    const markdownExample = [
        "```json",
        "{",
        "  \"firstName\": \"John\",",
        "  \"lastName\": \"Smith\",",
        "  \"age\": 25",
        "}",
        "```"
    ].join("\n")

    doSomething(markdownExample)  // Output/process the markdown somewhere
}
````````

With raw string literal feature, we don't need to escape anything:
````````js
const generateMarkdown = () => {
    const markdownExample = @````
        ```json
        {
          "firstName": "John",
          "lastName": "Smith",
          "age": 25
        }
        ```
        ````
    doSomething(markdownExample)  // Output/process the markdown somewhere
}
````````
#### (c) Regular expressions when string interpolation is also necessary

Without this proposal, the closest way to represent "raw" regex is this way:
`````js
new RegExp(String.raw`someRegex\b${processedSearchKeyword}\s*(?:\(?HKD\)?):?\s*`, "i")
`````
Even so, regex itself is already quite backslash heavy.

If the regex pattern itself contains several backticks and "${xxx}" to search for, it will be a bit error-prone to eyeball which backslashes are really in the regex and which are only for escaping characters at JavaScript side.

#### (d) XML/HTML/Source code embedding or generation

If we work on some HTML, XML, or even source code generators (e.g. JavaScript, Linux commands), we often need to include special characters (", ', $, ` at the same time). The ability to write them in unescaped manner with clear indentation will help improvement readability and maintainability.

If we work on some HTML, XML, or even source code generators (for e.g. JavaScript, Linux commands, PowerShell commands), we often need to include special characters (", ', $, ` at the same time). The ability to write them in unescaped manner with clear indentation will help improvement readability and maintainability.

One may argue the use of backtick is not frequent. But it is a was. Nowadays, many programming languages are adopting the backtick character just to avoid escaping double quotes, so backtick character is becoming more and more common.

Raw string literal may not solve all problems, but it can at least end the wild goose chase of choosing escape characters related to open and closing delimiter and interpolation delimiter.

#### (e) Unit tests involving MySQL query, Markdown, XML

Unit tests are also common places where we want to embed these contents in source codes instead of separate files.

As explained in (a) and (b), the raw string literal would improve clarity.

#### (f) A complex example involving nesting Markdown and JavaScript itself
````javascript
let promptForLLM = `
You are a AI assistant to give advice to programmers,
for example, given the code:
\`\`\`js
let s1 = "This is a\n"
  + "string across\n"
  + "multiple lines.\n"
let a = 1, b = 2
let s2 = "a + b = " + (a + b)
\`\`\`
you would output the advice:
\`\`\`\`markdown
## Advice
It's more readable to use template literal to replace
the string concatenation.

## Original code
\`\`\`js
let s1 = "This is a\n"
  + "string across\n"
  + "multiple lines.\n"
let a = 1, b = 2
let s2 = "a + b = " + (a + b)
\`\`\`

## Improved code
\`\`\`js
let s1 = String.dedent\`
  This is a
  string across
  multiple lines
  \`
let a = 1, b = 2
let s2 = \`a + b = \${a + b}\`
\`\`\`
\`\`\`\`
`
````

With proposed syntax:
``````text
let promptForLLM = @`````
    You are a AI assistant to give advice to programmers,
    for example, given the code:
    ```js
    let s1 = "This is a\n"
      + "string across\n"
      + "multiple lines.\n"
    let a = 1, b = 2
    let s2 = "a + b = " + (a + b)
    ```
    you would output the advice:
    ````markdown
    ## Advice
    It's more readable to use template literal to replace
    the string concatenation.

    ## Original code
    ```js
    let s1 = "This is a\n"
      + "string across\n"
      + "multiple lines.\n"
    let a = 1, b = 2
    let s2 = "a + b = " + (a + b)
    ```

    ## Improved code
    ```js
    let s1 = @``
      This is a
      string across
      multiple lines
      ``
    let a = 1, b = 2
    let s2 = `a + b = ${a + b}`
    ```
    ````
    `````
``````

## Relations to other proposals

### `String.dedent` proposal
TODO

### `String.cooked` proposal
- https://github.com/tc39/proposal-string-cooked/issues/7

Raw string literals should always give raw string，the current proposed `String.cooked` won't access `raw` property and cook the string, so the result may not as dev expect.

## Prior Arts

- [Here document](https://en.wikipedia.org/wiki/Here_document) (Shells, Perl, Ruby, PHP, D, Racket, etc.)
- C# [raw string literal](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/raw-string-literal)
- Swift [extended string delimiters](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/stringsandcharacters/#Extended-String-Delimiters)
- Rust [raw string literals](https://doc.rust-lang.org/reference/tokens.html#raw-string-literals)
- C++ [raw string literals](https://en.cppreference.com/w/cpp/language/string_literal#Raw_string_literals)

## Previous discussions/ideas
- https://es.discourse.group/t/raw-string-literals-that-can-contain-any-arbitrary-text-without-the-need-for-special-escape-sequences/1757
- https://github.com/tc39/proposal-string-dedent/issues/40
- https://github.com/tc39/proposal-string-dedent/issues/76
