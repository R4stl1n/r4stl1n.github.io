---
layout: post
title: OverTheWire - Natas 0-7
---

Site: http://overthewire.org/wargames/natas/ <br>
Level: 0-7<br>
Situation: Basic Web <br><br>

These challenges will be simple at first. I will try to make to make it as detailed as i can but some challenges might prevent it.

### Natas 0
- - - - - - -

This challenge is simple enough all that is required is to view the source of the site. Using our favorite browser we just right click and view source.

{% highlight bash %}
<div id="content">
You can find the password for the next level on this page.

<!--The password for natas1 is [OMITTED] -->
</div>
{% endhighlight %}

### Natas 1
- - - - - - -

Same as natas0 we need to get past the little issue of not being able to right click. To bypass this all we need to do is either use the view source hotkey or menu.

{% highlight bash %}
<div id="content">
You can find the password for the
next level on this page, but rightclicking has been blocked!

<!--The password for natas2 is [OMITTED] -->
</div>
{% endhighlight %}

### Natas 2
- - - - - - -

We view the source and the password isn't there however we see it includes a file called "pixel.png" in the image source. If directory listing is enabled in the apache config we should see what is in that directory.

![Embeded](/images/natas/screen1.png)
[FullSize](/images/natas/screen1.png)

If we look in users.txt we find our password.

{% highlight bash %}
# username:password
alice:BYNdCesZqW
bob:jw2ueICLvT
charlie:G5vCxkVV3m
natas3: [OMITTED]
eve:zo4mJWyNj2
mallory:9urtcpzBmH
{% endhighlight %}

### Natas 3
- - - - - - -

As before we start by viewing the source and we are presented by a message that says "Not even google will find it this time". For those who are familiar with how google spiders or web spiders in general work they look on the root address for a robots.txt. Robots.txt contains a set of "rules" so to speak for spiders to follow. This can include omitting directories. So let’s go to the "address/robots.txt".

![Embeded](/images/natas/screen2.png)
[FullSize](/images/natas/screen2.png)

We see a disallowed "secret" directory if we go in there we find a users.txt. With the following contents.

{% highlight bash %}
natas4:[OMITTED]
{% endhighlight %}

### Natas 4
- - - - - - -

Once we are inside natas 5 we are presented with the following message. 

{% highlight bash %}
Access disallowed. You are visiting from "" while authorized users should come only from "http://natas5.natas.labs.overthewire.org/"
{% endhighlight %}

When we click the refresh button it changes to the following.

{% highlight bash %}
Access disallowed. You are visiting from "http://natas4.natas.labs.overthewire.org/" while authorized users should come only from "http://natas5.natas.labs.overthewire.org/" 
{% endhighlight %}

We can tell that it is using our html headers to see where our src is coming from. We can get past this by simply changing our Referrer header that is sent with our request. There are a few ways to do this one can do it with burp suite etc. For us we will simply install a chrome plugin called "Referrer Control" and add a rule.

![Embeded](/images/natas/screen3.png)
[FullSize](/images/natas/screen3.png)

After adding the rule we refresh our page and we are given the following.

{% highlight bash %}
Access granted. The password for natas5 is [OMITTED] 
{% endhighlight %}.

### Natas 5
- - - - - - -

Natas5 presents us with.

{% highlight bash %}
Access disallowed. You are not logged in
{% endhighlight %}

If we view our source its empty and we aren’t aware of it checking for any referrer URL however. If we look as our resources tab in the built in debugger. We notice that the site is setting a cookie called "loggedin" and its set to 0. If this is a boolean check 0 = false and 1 = true. So we are going to modify this cookie. Yet again there are ways to do this but to keep it all in our browser. We install a plugin called "EditOurCookie" and we use it to change the value.

![Embeded](/images/natas/screen4.png)
[FullSize](/images/natas/screen4.png)

After we set the value and apply, we refresh the screen and we are given the following.

{% highlight bash %}
Access granted. The password for natas6 is [OMITTED]
{% endhighlight %}

### Natas 6
- - - - - - -

We are presented with an "input secret form" and when we enter garbage data we can see that it does indeed do a post and tells us we have the wrong secret. So to gain more information we click view the source and find the following.

{% highlight php %}
<?
    include "includes/secret.inc";
    if(array_key_exists("submit", $_POST)) {
        if($secret == $_POST['secret']) {
        print "Access granted. The password for natas7 is <censored>";
    } else {
        print "Wrong secret";
    }
    }
?>
{% endhighlight %}

We can see that it is pulling the $secret from a include file. Now if we aren’t forbidden from accessing the file we can get our string so navigate to "address"+/includes/secret.inc.

We are presented with the following.

{% highlight php %}
<?
$secret = "FOEIUWGHFEEUHOFUOIU";
?>
{% endhighlight %}

After entering the secret key into the form. We get the next message.

{% highlight bash %}
Access granted. The password for natas7 is [OMITTED]
{% endhighlight %}

### Natas 7
- - - - - - -

We start like we usually do and check the source we are immediately presented with the following message.

{% highlight bash %}
<!-- hint: password for webuser natas8 is in /etc/natas_webpass/natas8 -->
{% endhighlight %}

Telling us we will have to access the local file system somehow. What we do however is have two links "Home" and "About". Home turns our URL to this.

{% highlight bash %}
index.php?page=home
{% endhighlight %}

About changes it to the following.

{% highlight bash %}
index.php?page=about
{% endhighlight %}

When changes to a random file we get the following.

{% highlight bash %}
Warning: include(asdf): failed to open stream: No such file or directory in /var/www/natas/natas7/index.php on line 21

Warning: include(): Failed opening 'asdf' for inclusion (include_path='.:/usr/share/php:/usr/share/pear') in /var/www/natas/natas7/index.php on line 21
{% endhighlight %}

The following line tells us that we are trying to include a file called "asdf". This is a local file inclusion vulnerability.

{% highlight bash %}
Warning: include(asdf):
{% endhighlight %}

If we change it to the following.
{% highlight bash %}
index.php?page=/etc/natas_webpass/natas8
{% endhighlight %}

We are presented with the password.