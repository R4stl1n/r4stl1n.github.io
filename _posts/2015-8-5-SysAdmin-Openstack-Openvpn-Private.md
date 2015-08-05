---
layout: post
title: SysAdmin - Configure OpenVpn within Openstack To Private Network
---

Recently I have been digging deep into openstack and came across a personal
need to gain access to private network configured within openstack from my external
machine. Rather than giving each machine a floating ip. Below is the simple
network topology we will setup.

__Note: We want the vpn users to exist on 192.168.100.0/24__

![Embeded](/images/misc/openstack-private-network-diag.png)
[FullSize](/images/misc/openstack-private-network-diag.png)

This guide is going to assume the following.

* You already have a openstack cloud provider / personal lab.
* You have a public and private network with corresponding subnets.
__Note: The subnet can be using dhcp and have a gateway set it won't interfere.__
* You have already create instances on your private network and a instance on
your public network with a public ip associated with it to act as the firewall/vpn provider
* You are using ubuntu for your firewall/vpn distribution

##### Step 1
- - - - - - -
First thing that must be done is installing the required packages.

{% highlight bash %}
sudo apt-get install openvpn easy-rsa
{% endhighlight %}

##### Step 2
- - - - - - -
After the packages are installed we will now need to generate our certificates.
The first certificate we are going to generate is our RootCa. If you already
have a root ca generated you can utilize your own.

We create a easy-rsa folder in our openvpn folder and copy over the scripts.

{% highlight bash %}
ubuntu@firewall:~$ sudo mkdir /etc/openvpn/easy-rsa
ubuntu@firewall:~$ sudo cp -r /usr/share/easy-rsa/* /etc/openvpn/easy-rsa
{% endhighlight %}

Now we are going to modify the file titled "vars" in /etc/openvpn/easy-rsa/vars

{% highlight bash %}
ubuntu@firewall:~$ sudo nano /etc/openvpn/easy-rsa/vars
{% endhighlight %}

In this file you will want to modify the following variables. Within the file.
{% highlight bash %}
KEY_COUNTRY = Country of business code ex."US"
KEY_PROVINCE = Province / State of issuer ex. "Texas"
KEY_CITY = City of issuer ex. "Dallas"
KEY_ORG = Organization the certificate belongs to. ex."EvilCorp"
KEY_EMAIL = Administration email for certificate. ex. "admin@evilcorp.com"
KEY_CN = The common name for the cert ex. "XYZ Root CA"
KEY_NAME = The name of the certificate ex."XYZ ROOT CA"
KEY_OU = Organization Unity ex."Administration"
KEY_ALTNAMES = Alternative Name for the certificate "XYZ Root CAFORREAL"
{% endhighlight %}

Now that we have configured the variables we need to generate the certificate.
For ease of use we will also drop into a root terminal session. When running the
build-ca command you will be prompted for information.

{% highlight bash %}
ubuntu@firewall:~$ sudo bash
root@firewall:~# cd /etc/openvpn/easy-rsa
root@firewall:/etc/openvpn/easy-rsa# source vars
root@firewall:/etc/openvpn/easy-rsa# ./clean-all
root@firewall:/etc/openvpn/easy-rsa# ./build-ca
{% endhighlight %}

You should now have to files in the keys directory ca.crt and ca.key.

Next we need to generate our vpn server's certificates. You will be prompted for
input.

__Note: You can change "MyVpnServerCert" to whatever name you want__
__Note: Make sure when it asks if you want to sign the certificate that you sign it
or else there will be problems__

{% highlight bash %}
root@firewall:/etc/openvpn/easy-rsa# ./build-key-server MyVpnServerCert
{% endhighlight %}

Now that we have that configured we need to generate our Diffie-Hellman parameters
file.

{% highlight bash %}
root@firewall:/etc/openvpn/easy-rsa# ./build-dh
{% endhighlight %}

Finally the last thing we need to generate a shared secret file to help prevent
against DDOS and other Nasty things.

{% highlight bash %}
root@firewall:/etc/openvpn/easy-rsa# openvpn --genkey --secret ta.key
{% endhighlight %}

Now we copy these files we have generated to the /etc/openvpn/certs folder.

{% highlight bash %}
root@firewall:/etc/openvpn/easy-rsa# mkdir /etc/openvpn/certs
root@firewall:/etc/openvpn/easy-rsa/keys# cd keys
root@firewall:/etc/openvpn/easy-rsa/keys# cp ca.crt /etc/openvpn/certs/
root@firewall:/etc/openvpn/easy-rsa/keys# cp MyVpnServerCert.crt /etc/openvpn/certs/
root@firewall:/etc/openvpn/easy-rsa/keys# cp MyVpnServerCert.key /etc/openvpn/certs/
root@firewall:/etc/openvpn/easy-rsa/keys# cp dh2048.pem /etc/openvpn/certs/
root@firewall:/etc/openvpn/easy-rsa/keys# cp ta.key /etc/openvpn/certs/
root@firewall:/etc/openvpn/easy-rsa/keys# cd ../
{% endhighlight %}

##### Step 3
- - - - - - -

Now we need to configure the openvpn server itself. First we will copy the example
configuration file for openvpn to our openvpn folder.

{% highlight bash %}
root@firewall:/etc/openvpn/easy-rsa# cd /etc/openvpn
root@firewall:/etc/openvpn# gunzip -d server.conf.gz
{% endhighlight %}

A file named "server.conf" should be in /etc/openvpn.
There is quite a few changes we must make to this.
{% highlight bash %}
root@firewall:/etc/openvpn# nano server.conf
{% endhighlight %}

**First we have to set the correct protocol**

By default it is set to udp which is fine for most networks. However certain
providers have a problem with udp tunneled traffic dropping. Because of this
we set it to tcp.

Find the line that states the following.

{% highlight bash %}
proto udp
{% endhighlight %}

Change it to the following

{% highlight bash %}
proto tcp
{% endhighlight %}

**Next we have to set the correct certificate**

Find the following two lines.

{% highlight bash %}
cert server.crt
key server.key
{% endhighlight %}

Change them to the following

{% highlight bash %}
cert certs/MyVpnServerCert.crt
key certs/MyVpnServerCert.key
{% endhighlight %}

**Now we need to set the correct Diffie-Hellman parameters.**

Find the line that states

{% highlight bash %}
dh dh1024.pem
{% endhighlight %}

Change it to the following

{% highlight bash %}
dh dh2048.pem
{% endhighlight %}

**Next we need to configure the vpn client address that is issues out.**

Locate the following line

{% highlight bash %}
server 10.8.0.0 255.255.255.0
{% endhighlight %}

Replace it with the following

{% highlight bash %}
server 192.168.100.0 255.255.255.0
{% endhighlight %}

**Next we need to configure our "route" to the 192.168.1.0/24 network.**

Find the following line

{% highlight bash %}
;push "route 192.168.20.0 255.255.255.0"
{% endhighlight %}

Under that line add the following line.

{% highlight bash %}
push "route 192.168.1.0 255.255.255.0"
{% endhighlight %}

**Next just need to enable the shared secret**

Find the following line.

{% highlight bash %}
;tls-auth ta.key 0
{% endhighlight %}

Change it to the following

{% highlight bash %}
tls-auth certs/ta.key 0
{% endhighlight %}

**Finally we just enable logging to help with our sanity**

Find the following line

{% highlight bash %}
;log openvpn.log
{% endhighlight %}

Change it to the following

{% highlight bash %}
log openvpn.log
{% endhighlight %}

Now that the openvpn server configuration is done we just need to start it.

{% highlight bash %}
service openvpn start
{% endhighlight %}

##### Step 5
- - - - - - -

To finish the server configuration we need to allow for our vpn server to act
as a router.

**To enable the functionality immediately we need to do the following.**

Run the following command to enable ip_forwarding

{% highlight bash %}
root@firewall:/etc/openvpn# echo "1" > /proc/sys/net/ipv4/ip_forward
{% endhighlight %}

Now the iptables to allow our packets to be forward to the correct destination.

__Note: the eth1 interface is important in the POSTROUTING if you pick the wrong
interface you will not be able to communicate with the private network__

{% highlight bash %}
root@firewall:/etc/openvpn# iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
root@firewall:/etc/openvpn# iptables -A FORWARD -s 192.168.100.0/24 -j ACCEPT
root@firewall:/etc/openvpn# iptables -A FORWARD -j REJECT
root@firewall:/etc/openvpn# iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth1 -j MASQUERADE
{% endhighlight %}

**Now we just need to make it permanent.**

First to ensure that ip forwarding is always enabled we need to modify the sysctl configuration.

{% highlight bash %}
root@firewall:/etc/openvpn# nano /etc/sysctl.conf
{% endhighlight %}

Find the following line
{% highlight bash %}
#net.ipv4.ip_forward=1
{% endhighlight %}

Change it to the following

{% highlight bash %}
net.ipv4.ip_forward=1
{% endhighlight %}

**Finally we persist the iptables data.**

We achieve this by doing it the lazy way. By modifying our /etc/rc.local file.

{% highlight bash %}
root@firewall:/etc/openvpn# nano /etc/rc.local
{% endhighlight %}

Find the following line.

{% highlight bash %}
exit 0
{% endhighlight %}

Before that line add the following

{% highlight bash %}
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -s 192.168.100.0/24 -j ACCEPT
iptables -A FORWARD -j REJECT
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth1 -j MASQUERADE
{% endhighlight %}

Save the file and thats it server is configured and will survive a reboot.

##### Step 6
- - - - - - -

For each user we wish to allow onto our vpn we will have to generate a separate
certificate signed by our vpn certificate. You will be prompted for information.

__Note: You can change TestClient to whatever you want.__
__Note: Make sure to sign the certificate.__
{% highlight bash %}
ubuntu@firewall:~$ sudo bash
root@firewall:~# cd /etc/openvpn/easy-rsa
root@firewall:/etc/openvpn/easy-rsa# source vars
root@firewall:/etc/openvpn/easy-rsa# ./build-key TestClient
root@firewall:/etc/openvpn/easy-rsa# exit
{% endhighlight %}

Now we copy them to a folder in our home directory

{% highlight bash %}
ubuntu@firewall:~$ mkdir ~/TestClient
ubuntu@firewall:~$ cd /etc/openvpn/easy-rsa/keys
ubuntu@firewall:~$ sudo cp ca.crt ~/TestClient
ubuntu@firewall:~$ sudo cp TestClient.crt ~/TestClient
ubuntu@firewall:~$ sudo cp TestClient.key ~/TestClient
ubuntu@firewall:~$ sudo cp ta.key ~/TestClient
{% endhighlight %}

Now we need to copy the sample client configuration file and modify it.

{% highlight bash %}
ubuntu@firewall:~$ cd ~/TestClient
ubuntu@firewall:~$ cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf .
{% endhighlight %}

{% highlight bash %}
ubuntu@firewall:~$ nano client.conf
{% endhighlight %}

**First we set the correct protocol**

Find the following line

{% highlight bash %}
proto udp
{% endhighlight %}

Change it to the following

{% highlight bash %}
proto tcp
{% endhighlight %}

**Next we have to set the correct remote address**

Find the following

{% highlight bash %}
remote my-server-1 1194
{% endhighlight %}

Change it to the following

{% highlight bash %}
remote YOUR-SERVER-PUBLIC-IP 1194
{% endhighlight %}

**After that we have to set the correct certificates**

Find the following lines

{% highlight bash %}
ca ca.cert
cert client.crt
key client.key
{% endhighlight %}

Change them to the following

{% highlight bash %}
ca ca.cert
cert TestClient.crt
key TestClient.key
{% endhighlight %}

**Finally we enable the shared secret**

Find the following line

{% highlight bash %}
;tls-auth ta.key 1
{% endhighlight %}


Change it to the following

{% highlight bash %}
tls-auth ta.key 1
{% endhighlight %}

After that the client configuration files are ready to go.

That's it if everything works out well the openvpn server will be configured.
Permissions will have to be changed on the files in the ~/TestClient folder.

{% highlight bash %}
ubuntu@firewall:~$ sudo chown -R user:user ~/TestClient
{% endhighlight %}
