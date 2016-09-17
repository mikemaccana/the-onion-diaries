# Building a "Proof of Concept" Onion Site
## …by using a rewrite-proxy for a normal, cleartext web site

## Goal

Let's build an Onion site which serves proxied-and-rewritten content for a single port (port 80) of the BBC website.

The BBC is (currently) suitable for exemplar experimentation because it does not use HTTPS to protect its content, plus it's unlike to complain from a corporate standpoint and has no concept of logged-in users to be interfered with.

Once we are happy with this process, the next document can build upon this and describe how to do the same for a multi-tcp-port and multi-domain (eg: CDN-enabled) website.

## You Will Need

* Basic Linux sysadmin skills
* A fresh Ubuntu Server 16.04.1 instance, or similar
  * I use VirtualBox on a laptop for convenience
  * A virtual host or AWS instance would also be okay
* Outbound network access
* Maybe a couple of hours of free time
* Coffee

## Notes

#### Networking

Don't fret about firewalls or lack of inbound internet access; one of the joys of Onion sites is that they only need or use *outbound* network access.

Tor onions work because any hypothetically "inbound" traffic is received by coming backwards up a connection which was originally established *outbound*.

This also has the nice side effect that you can lock your onions in a DMZ "enclave", restricted to permit outbound internet access only, and limiting any other connectivity.

#### Production

I recommend not using this process for anything other that Proof of Concept of your own website.

Certainly I consider the *rewrite HTML on the fly* approach which we a re describing here, to be greatly inferior to fixing your CMS to manage access from multiple and separate domainnames elegantly.

However, this is a nice, simple taster of *how easy it is to onionify a site*.

## Installation Steps

This is probably not as optimal as it could be, but it is tested and works.

#### Build your System

Install Ubuntu with "standard system utilities" and apply all patches; we will do everything else manually.

You might want to install sshd (eg: package `openssh-server`) so that you have terminal cut-and-paste; I shall leave that undocumented, but please maintain security appropriately.

#### Build `mitmproxy`

There is a version of mitmproxy available in the Ubuntu APT repositories, but it is old, out of date, and wrong with respect to the documentation that you will find on the web.

So we shall build it manually.  The commands I am typing are as follows:

```sh
sudo -i # become root

apt-get install aptitude # if you can do without this, you're a hero

aptitude install libffi-dev libjpeg-dev libssl-dev libxml2-dev libxslt1-dev libyaml-dev python-dev python-pip zlib1g-dev

pip install --upgrade pip

pip install mitmproxy

mitmproxy --version
```

This should give you `mitmproxy 0.17` if everything works okay.

#### Install Tor

This is all nicely documented at https://www.torproject.org/docs/debian.html.en

Select:

* Ubuntu Xenial Xerus
* Tor
* Stable

...in the menu options, and follow the instructions which are provided. It should basically involve editing 1 file to add some APT configuration, plus executing 4 commands.

#### Configure an Onion Site

Edit `/etc/tor/torrc` to include the following in the location-hidden services section.

```sh
HiddenServiceDir /var/lib/tor/bbc-onion/
HiddenServicePort 80  localhost:20080
```

#### Restart Tor

Do:

```sh
/etc/init.d/tor restart
```

#### Collect your Onion Site name

Do:

```sh
cd /var/lib/tor/bbc-onion
cat hostname
```

...and remember the result; it will look a bit like `abcdefghijklmnop.onion` - we shall use that as the example, but make sure to use your own one when you try this.

#### Enable `mitmproxy`

Create a shell script, call it `run-proxy.sh`, making the necessary edits as documented in the file:

```sh
#!/bin/sh

# IMPORTANT (1/2): Edit this to match your onion site
ONION=abcdefghijklmnop.onion

# IMPORTANT (2/2): uncomment one of these two lines
#SITE=bbc.com   # uncommment this if you live outside the UK
#SITE=bbc.co.uk # uncomment this if you live inside the UK

# check
if [ "x$SITE" = x -o "x$ONION" = xabcdefghijklmnop.onion ] ; then
    echo you did not read the instructions in the script
    exit 1
fi

# escape dots in regexps
Dotify() {
    echo $1 | sed -e 's/\./\\./g'
}

# we are building a 1:1 mapping for only the 'www' site, not the whole TLD
SITE=www.$SITE
ONION=www.$ONION

# sub-edit the regular expressions
SITE_RE=`Dotify $SITE`
ONION_RE=`Dotify $ONION`

# tell the user
echo "when you press return, onion access for $SITE will become available on:"
echo http://$ONION
echo "...press return to launch the proxy. Use 'q' to exit the proxy."
read junk

# run
exec mitmproxy \
    -p 20080 \
    -R http://${SITE} \
    --anticache \
    --replace ":~hq .:${ONION_RE}:${SITE}" \
    --replace ":~hs .:${SITE_RE}:${ONION}" \
    --replace ":~bs . ~t \"application/json\":${SITE_RE}:${ONION}" \
    --replace ":~bs . ~t \"application/x-javascript\":${SITE_RE}:${ONION}" \
    --replace ":~bs . ~t \"text/css\":${SITE_RE}:${ONION}" \
    --replace ":~bs . ~t \"text/html\":${SITE_RE}:${ONION}" \
    --replace ":~s:${SITE_RE}:${ONION}"
```

Do the following, and follow the instructions
```sh
chmod u+x run-proxy.sh
./run-proxy.sh
```


### Connect to the Onion Site

Open TorBrowser and connect to `http://www.abcdefghijklmnop.onion` (amend this for your own onion site)

You should see a (slightly flaky) version of the BBC website.