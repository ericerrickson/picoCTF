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
 `Disallow: /admin.phps`  
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
OK, now we are getting somewhere.



