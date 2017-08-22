---
title: Generating Key Pairs in PHP
---

In order to test an installation of ProFTPd that uses a PHP backend to populate a MySQL database for key authenticatd SFTP access I need to programatically generate keys and then get the public key in [RFC4716](https://tools.ietf.org/html/rfc4716) format. This format is required because that is what the SFTP module of ProFTPD requires. The backend is PHP and the testing framework is [CodeCeption](https://codeception.com). 

### PHPSecLib
The excellent [PHPSecLib](https://github.com/phpseclib/phpseclib) library would provide everything I need for this, but at the moment the [specific functionality](https://github.com/phpseclib/phpseclib/blob/master/phpseclib/Crypt/RSA/Keys/PuTTY.php) I need is only in master and master has a note on it saying "Do not use in production"... so I guess I need to find another way of doing things. 

### PHP OpenSSL functions
The [PHP openssl functions](http://php.net/manual/en/ref.openssl.php) seem to provide what we need, so: 
```php?start_inline=1
$key = openssl_pkey_new (array('private_key_bits' => 2048, "private_key_type" => OPENSSL_KEYTYPE_RSA));
if (false === $key) { // Check for errors - first time round this revealed that I hadn't installed openssl in my container. 
  $error = openssl_error_string();
  while (false !== $error) {
    echo "$error\n";
    $error = openssl_error_string();
  }
  exit(0);
}
var_dump($key);
// resource(1) of type (OpenSSL key)
```

The first time I tried I got:
```
error:02001002:system library:fopen:No such file or directory
error:2006D080:BIO routines:BIO_new_file:no such file
error:0E064002:configuration file routines:CONF_load:system lib
error:02001002:system library:fopen:No such file or directory
error:2006D080:BIO routines:BIO_new_file:no such file
error:0E064002:configuration file routines:CONF_load:system lib
error:02001002:system library:fopen:No such file or directory
error:2006D080:BIO routines:BIO_new_file:no such file
error:0E064002:configuration file routines:CONF_load:system lib
error:02001002:system library:fopen:No such file or directory
error:2006D080:BIO routines:BIO_new_file:no such file
error:0E064002:configuration file routines:CONF_load:system lib
```
Which I eventually tracked down to not having openssl installed. Easily fixed in Alpine, by running `apk --no-cache install openssl`. Newer versions of alpine are switching to libressl, so substitute that in for Edge or v4 when it comes. 

However, although we now have a key pair in the `$key` variable it seems that there is no sensible way of extracting the public components. The recommended [openssl_pkey_get_public function](http://php.net/manual/en/function.openssl-pkey-get-public.php) seems to require a certificate, not a key resource and that just seems like unessersary messing around.

### Exec ssh-keygen
This feels like a dirty hack, but at the moment it is the only way I've found to do what I want to do. 
In the shell we can do this
```sh
## Generate a new keypair with no passphrase
ssh-keygen -t rsa -b 2048 -f /tmp/key -N '' -C 'Keypair for testing'
## Output to check it is OK
cat /tmp/key
## Output the public key in the right format. 
ssh-keygen -f /tmp/key  -e -m RFC4716
## Output the finger print
ssh-keygen -l -f /tmp/key
```

And in PHP we can do something like this: 
```php?start_inline=1
$keyFile = tempnam(sys_get_temp_dir(), 'key');
unlink($keyFile);
exec('ssh-keygen -t rsa -b 2048 -f "'.$keyFile.'" -N "" -C "Generated keys for testing" -q 2>&1', $output, $returnVal);
if (0 !== $returnVal) {
    throw new \Exception("Failed to generate key:\n".implode("\n", $output));
}
$private1 = file_get_contents($keyFile);
unset($output);
exec('ssh-keygen -e -f '.$keyFile.' -m RFC4716 2>&1', $output, $returnVal);
if (0 !== $returnVal) {
    throw new \Exception("Failed to get public key:\n".implode("\n", $output));
}
$public1 = implode("\n", $output);
unset($output);
exec('ssh-keygen -l -f '.$keyFile.' 2>&1', $output, $returnVal);
if (0 !== $returnVal) {
    throw new \Exception("Failed to get public key:\n".implode("\n", $output));
}
$split = explode(" ", $output[0]);
list($fingerprint_hash, $fingerprint) = explode(":",$split[1]);
if ('SHA256' === $fingerprint_hash) {
    $fingerprint1 = base64_decode($fingerprint);
} elseif ('MD5' === $fingerprint_hash) {
    $fingerprint1 =  hex2bin($fingerprint);
}
unset($output);
unlink($keyFile);
unlink("{$keyFile}.pub");
```
Which will result in `$private1`, `$public1` and `$fingerprint1` containing all the relevant things. As a bonus the fingerprint is decoded to its pure form rather than hex as string or base64 encoded. 
