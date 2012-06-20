# A basic Clojure web app using Jetty behind Apache

This is the simplest example I could come up with for getting a Clojure web app running under Jetty, proxied by Apache.  It's split up into three main sections:

  * [Getting Clojure and Jetty working together via Ring](https://github.com/ddellacosta/Clojure-under-Jetty-and-Apache#head1)
  * [Setting up Jetty with Apache HTTPD](https://github.com/ddellacosta/Clojure-under-Jetty-and-Apache#head2)
  * [Extra: getting Jetty running via a init script.](https://github.com/ddellacosta/Clojure-under-Jetty-and-Apache#head3)

I've tested this on my slicehost Gentoo Linux setup. It's been a good number of years since I've done much of anything in the Java world, so I will warn you that I may be neglecting configuration relating to optimization or security with regards to my Jetty setup.  Accordingly, please do contact me or fork this and send a [pull request](https://help.github.com/articles/using-pull-requests) if you want to add something as far as setting up Jetty or the Java/Clojure environment the "right" way (for example, I have no idea if it would be best to package Clojure or other jars with Jetty globally, or using the uberwar as I'm doing in this tutorial).  I'm very much open to and interested in improving this guide.

Please note that, where I say "<your favorite editor>" below in the shell examples, it is meant for you to substitute that with the name of your preferred editor, such as emacs, vim, nano, etc.  I should also add that this tutorial assumes familiarity with moving about your file system and editing files on the command line, knowledge of the structure of Jetty's XML configuration files and Apache's configuration files, and basic familiarity with Clojure syntax.  That said, if anything is particularly unclear, please drop me a line.

The software versions I'm using:

  * Java: OpenJDK Runtime Environment (IcedTea6 1.11.3) (Gentoo build 1.6.0_24-b24)
  * Jetty (http://www.eclipse.org/jetty/): 8.1.4.v20120524
  * Clojure (http://clojure.org/): 1.4.0
  * Leiningen (https://github.com/technomancy/leiningen): 2.0.0-preview6
  * Lein-Ring (https://github.com/weavejester/lein-ring): 0.7.1
  * Ring (https://github.com/ring-clojure/ring): 1.1.1
  * Apache HTTP Server (http://httpd.apache.org/): 2.2.22

## <a name="#head1" />Getting Clojure and Jetty working together via Ring

First of all, this tutorial assumes you've got your Java environment set up and working.

Next, make sure you've got Leiningen set up properly (https://github.com/technomancy/leiningen).  I had a pretty minimal lein setup for this tutorial with just the lein-ring plugin installed:

```clojure
; ~/.lein/profiles.clj
{:user
 {:plugins [[lein-ring "0.7.1"]]}}
```

Then, I created a new project using lein:

```shell
$ lein new web_test
```

In the project.clj file, I am using Clojure 1.4.0, and I added the ring and ring-jetty-adapter dependencies as well...and at the end, one line for a ring handler:

```shell
$ <your favorite editor> web_test/project.clj
```

```clojure
(defproject web_test "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.4.0"]
                 [ring/ring-core "1.1.1"]
                 [ring/ring-jetty-adapter "1.1.1"]]
  :ring {:handler web-test.core/handler})
```

Then in src/web_test/core.clj, I defined the handler:

```clojure
(ns web-test.core)

(defn handler [request]
  {:status 200
   :headers {"Content-Type" "text/html"}
   :body "Hello World"})
```

(All of the ring setup I pulled directly from the ring wiki page here: https://github.com/ring-clojure/ring/wiki/Getting-Started)

With that set up, I was ready to generate a .war file with ring.  I tried with the default `lein ring war` option, but I got complaints from Jetty that it was missing Java libraries (Clojure, for example...), so I tried again with the 'uberwar' option, as that includes all necessary libraries.  That worked:

```shell
$ lein ring uberwar
```

This creates a .war file, and puts it in a directory called target/ in the root of your app.  Copy this .war file to the webapps directory where you unpacked Jetty. In my case:

```shell
$ cp target/web_test-0.1.0-SNAPSHOT-standalone.war ../../jetty-distribution-8.1.4.v20120524/webapps/
```

Now, change directory to your Jetty install, and to get a working set of default configuration options, just copy the file contexts/test.xml (which comes packaged with Jetty by default) to a new file:

```shell
$ cp contexts/test.xml contexts/web_test.xml
```

Open it up:

```shell
$ <your favorite editor> contexts/web_test.xml
```

Find the line:

```xml
<Set name="war"><SystemProperty name="jetty.home" default="."/>/webapps/test.war</Set>
```

And edit this to match the war file you just copied over to the webapps/ directory.  In my case:

```xml
<Set name="war"><SystemProperty name="jetty.home" default="."/>/webapps/web_test-0.1.0-SNAPSHOT-standalone.war</Set>
```

Then, a bit further down, comment out or delete this line:

```xml
<Set name="overrideDescriptor"><SystemProperty name="jetty.home" default="."/>/contexts/test.d/override-web.xml</Set>
```

(Compare to the contexts/web_test.xml file in this repo for changes if confused.)

That's it.  At this stage, we'll test that things are working by starting up Jetty (from root of Jetty app):

```shell
$ java -jar start.jar
```

Open up a browser and head to

    http://<your IP or hostname>:8080/web_test-0.1.0-SNAPSHOT-standalone/

It's a bit verbose, but we're going to put it all behind an Apache proxy in a bit.

Everything beyond this point is Apache and Linux configuration.  If getting a Clojure app running on Jetty is all you were after, you can stop here.

## <a name="#head2" />Setting up Jetty with Apache HTTPD

We want our Clojure app to be there when you go to our domain. So we'll set up Apache as a proxy for our Jetty web app.

This more or less exactly follows the ["Configuring Apache"](http://wiki.eclipse.org/Jetty/Tutorial/Apache#Configuring_Apache) and ["Configuring mod_proxy"](http://wiki.eclipse.org/Jetty/Tutorial/Apache#Configuring_mod_proxy) sections from the Jetty documentation linked to below.

To start with, add these two lines to your Apache config.  For me this was in /etc/apache2/httpd.conf, at the bottom of a long list of other LoadModule calls:

    LoadModule proxy_module modules/mod_proxy.so
    LoadModule proxy_http_module modules/mod_proxy_http.so  # http://ubuntuforums.org/showthread.php?t=1434872

Then, add these lines to your vhost (or wherever your domain is set up...again, I'll assume you are familiar with your Apache setup):

    ProxyRequests Off
    ProxyVia Off
    ProxyPreserveHost On

    <Proxy *>
      AddDefaultCharset off
      Order deny,allow
      Allow from all
    </Proxy>

    ProxyPass / http://127.0.0.1:8080/web_test2-0.1.0-SNAPSHOT-standalone/

The Jetty Apache docs include the following lines, but I couldn't get it working, so I commented them out. I've provided them in case you notice the discrepancy in how I've configured things compared to the Jetty docs and wonder what's up, and in case you want to give it a shot. But the chunk of Apache config is not necessary.

    LoadModule status_module modules/mod_status.so  # (I assume you need this enabled, but it didn't make a difference with my setup.)

    ProxyStatus On
    <Location /status>
      SetHandler server-status
      Order Deny,Allow
      Allow from all
    </Location>

At this point, you should be ready to go.  Make sure Jetty is running, and restart Apache.  Go to your domain, and you should see your "Hello World" again.

## <a name="#head3" />Extra: getting Jetty running via a init script. 

If you want to set this up in a more permanent way, you will probably prefer to use a script to start up and shut down Jetty, like how most services running on Linux work. In my case, I'm using the Gentoo distribution, so I wanted something nice in my /etc/init.d folder to start and stop, and to configure as a default service, just like how Apache is set up (*TODO: configure for Gentoo rc-update so I can have this loaded by default*).  Please note that your *nix may be set up quite different (in particular, where the startup scripts are located).  However, the steps will probably be pretty similar.

Now, for whatever reason Jetty isn't available as a package within Gentoo's emerge package management system on my server.  Luckily the Jetty package comes with some nice scripts available in the bin/ directory (which unfortunately aren't conformant to Gentoo's format...see my TODO).  I copied the one for *nix systems to my /etc/init.d directory:

``` shell
# from the root of the Jetty install directory
$ cp bin/jetty.sh /etc/init.d/
```

The top of that file (bin/jetty.sh) provides a very detailed description of the options that are available for configuration. This includes which files it tries to read on startup, as well as a listing of relevant environment variables.

It states that it reads from a file called /etc/default/jetty on initialization.  I also noted that by default the script tries to find the jetty install directory in /opt/jetty.  However, I preferred where I had it set up, so I added one variable to the file /etc/default/jetty:

``` shell
JETTY_HOME=/var/www/jetty-distribution-8.1.4.v20120524
```

Now, I can run

```shell
$ /etc/init.d/jetty.sh start
```

## Documentation which was especially useful in writing this:

  * https://github.com/ring-clojure/ring/wiki/Getting-Started
  * http://www.enavigo.com/2008/08/29/deploying-a-web-application-to-jetty/
  * http://wiki.eclipse.org/Jetty/Tutorial/Apache
  * http://ubuntuforums.org/showthread.php?t=1434872 (this saved me after half an our of stumbling around; I didn't realize I needed mod_proxy_http!)

## TODO:

  * Get the init script configured with Gentoo's system so that I can have Jetty start up automatically on boot (http://www.gentoo.org/doc/en/handbook/handbook-x86.xml?part=2&chap=4#doc_chap2).

## License

Copyright &copy; 2012 Dave Della Costa

Distributed under the MIT License (http://dd.mit-license.org/)
