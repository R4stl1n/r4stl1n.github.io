---
layout: post
title: Programming - Compiling HxSSL For Win 10
---

Recently I have been getting into the Haxe programming language. One of the
useful parts about the language is its ability to target a wide range of
individual platforms. However it still requires the libraries that are used
to have native builds for each platform.

A side project of mine required openssl to be used to interact with https sites.
Unfortunately the only way to do this is to incorporate HxSSL. Usually an
individual would only have to do "haxelib install hxssl" however the current
version in the haxelib repository has a nasty bug that disables the normal "http" usage of haxe.http. So you need to use the dev branch of HxSSL.

The following things are assumed.

* You are running windows 10
* You have already installed haxe
* You have installed either the "Community" edition of Visual Studio 2013 or
the "Express Desktop" version.
* You have git-scm, cmder or equivalent installed
* You have removed any previous installation of hxssl

##### Step 1
- - - - - - -

First thing we want to do is grab a recent copy of the HxSSL project.
You can grab it at the following url https://github.com/tong/hxssl .
Or by copy and pasting the following into a command prompt.

{% highlight bash %}
git clone https://github.com/tong/hxssl.git
{% endhighlight %}

##### Step 2
- - - - - - -

Before we can compile hxssl we must first compile openssl. At the time of writing
the version of openssl packaged with HxSSL is (1.0.2a). Open the openssl.xml file
located in hxssl/openssl/project/buildfiles in your preferred editor.

Locate the following two lines in between the "<files id="openssl" > </files>"
{% highlight xml %}
<compilerflag value="-I../../include" />
<compilerflag value="-I../../include/openssl" />
{% endhighlight %}

Underneath the two lines add the following three lines.
{% highlight xml %}
<compilerflag value="-I./" />
<compilerflag value="-Iinclude" />
<compilerflag value="-Iinclude/openssl" />
{% endhighlight %}

_Yes below is the same thing just different location in the file_

Now locate the following two lines Locate the following two lines in between the "<files id="crypto" > </files>"

{% highlight xml %}
<compilerflag value="-I../../include" />
<compilerflag value="-I../../include/openssl" />
{% endhighlight %}

Underneath the two lines add the following three lines.
{% highlight xml %}
<compilerflag value="-I./" />
<compilerflag value="-Iinclude" />
<compilerflag value="-Iinclude/openssl" />
{% endhighlight %}

Now navigate to the openssl/tools folder and run the following command.
{% highlight bash %}
haxe compile.hxml
{% endhighlight %}

When that command completes navigate to openssl/project and run the following command.
{% highlight bash %}
neko build.n
{% endhighlight %}

If everything was done correctly libopenssl.lib should be created in lib/Windows/libopenssl.lib

##### Step 3
- - - - - - -

Now that we have compiled openssl we need to compile HxSSL. To do this we must
first make some changes in the build.xml file located in the hxssl/src/ folder.

In the file locate the following two lines.

{% highlight xml %}
<compilerflag value="-I../openssl/project/include/" />
<compilerflag value="-I../openssl/project/include/openssl" />
{% endhighlight %}

After those two lines insert the following.

_Keep in mind that the openssl directory might change if a different version is bundled with HxSSL in the future_
{% highlight xml %}
<compilerflag value="-I../openssl/project/unpack/openssl-1.0.2a/include/openssl" />
<compilerflag value="-I../openssl/project/unpack/openssl-1.0.2a/include" />
{% endhighlight %}

Next we need to add the correct library. Locate the following line in the file.

{% highlight xml %}
<lib name="Ws2_32.lib" if="windows" />
{% endhighlight %}

Before this line add the following line.

{% highlight xml %}
<lib name="..\openssl\lib\Windows\libopenssl.lib" if="windows" />
{% endhighlight %}

After that run the following command in the src folder.

{% highlight bash %}
haxelib run hxcpp build.xml
{% endhighlight %}

If everything is correct it will create the hxssl.lib file.

##### Step 4
- - - - - - -

Now all that is left is to install the newly compiled package into haxelib.

To do this place the HxSSL folder where you wish for it to be permanently stored.
Then run the following command.

{% highlight bash %}
haxelib dev hxssl <DirectoryToHxSSLFolder>
{% endhighlight %}

##### Step 5
- - - - - - -

Finally we just test the installation. Create a file called TestHttps.hx and
copy the following code into it.

{% highlight haxe %}
import sys.ssl.Socket;
import sys.net.Host;

class TestHttps {
	static inline var URL = "www.google.com";
	static inline var PORT = 443;

	static public function main(): Void {
		var s = new sys.ssl.Socket();
		s.validateCert = false;
		s.connect(new Host(URL), 443);

		trace("Connected");
		s.write("GET / HTTP/1.1\r\n
							Host: HxSSL\r\n
							Accept: */*\r\n
							\r\n");
		trace(s.read());
		s.close();

	}

}
{% endhighlight %}

Then run the following command.

{% highlight bash %}
haxe -main TestHttps.hx -cpp test -lib hxssl TestHttps.hx
{% endhighlight %}

If all goes well a TestHttps.exe will be created in the test folder. That is all
it takes to get it compiling.
