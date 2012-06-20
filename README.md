# A basic Clojure web app using Jetty and Apache

This is the simplest example I could come up with for getting a Clojure web app running under Jetty with Apache. I tested this on my slicehost Gentoo Linux setup. It's been a good number of years since I've done much of anything in the Java world, so I will warn you that I may be leaving out issues relating to optimization or security with regards to my Jetty setup.  Accordingly, please do contact me or fork this and send a pull request (https://help.github.com/articles/using-pull-requests) if you want to add something relating to setting up Jetty or the Java environment the "right" way (for example, I have no idea if it would be best to package Clojure with Jetty globally, or using the uberwar as described in this tutorial).

The software versions I'm using:

  * Java: OpenJDK Runtime Environment (IcedTea6 1.11.3) (Gentoo build 1.6.0_24-b24)
  * jetty (http://www.eclipse.org/jetty/): 8.1.4.v20120524
  * leiningen (https://github.com/technomancy/leiningen): 2.0.0-preview6
  * lein-ring (https://github.com/weavejester/lein-ring): 0.7.1
  * clojure (http://clojure.org/): 1.4.0
  * ring (https://github.com/ring-clojure/ring): 1.1.1
  * apache httpd (http://httpd.apache.org/): 2.2.22

## Getting Clojure and Jetty working together via Ring

First of all, this tutorial assumes you've got your Java environment set up and working.

Next, make sure you've got lein set up properly (https://github.com/technomancy/leiningen).  I had a pretty minimal lein setup for this tutorial with just the lein-ring plugin installed:

    ; ~/.lein/profiles.clj
    {:user
     {:plugins [[lein-ring "0.7.1"]]}}

Then, I created a new project using lein:

    $ lein new web_test

I am using Clojure 1.4.0 for this, and I added added the ring and ring-jetty-adapter dependencies.

    $ <your favorite editor> web_test/project.clj

    (defproject web_test "0.1.0-SNAPSHOT"
      :description "FIXME: write description"
      :url "http://example.com/FIXME"
      :license {:name "Eclipse Public License"
                :url "http://www.eclipse.org/legal/epl-v10.html"}
      :dependencies [[org.clojure/clojure "1.4.0"]
                     [ring/ring-core "1.1.1"]
                     [ring/ring-jetty-adapter "1.1.1"]]

And at the end, one line for a ring handler:

      :ring {:handler web-test.core/handler})

Then in src/web_test/core.clj, I defined the handler:

    (ns web-test.core)

    (defn handler [request]
      {:status 200
       :headers {"Content-Type" "text/html"}
       :body "Hello World"})

(All of the ring setup I pulled directly from the ring wiki page here: https://github.com/ring-clojure/ring/wiki/Getting-Started)

With that set up, I was ready to generate a war with ring.  I tried with the default war, but I got complaints from Jetty that it was missing java libraries (Clojure, for example...), so I tried again with the 'uberwar' option:

    $ lein ring uberwar

This creates a war, and puts it in a new target/ directory in the root of your app.  Copy this war to the webapps directory where you unpacked Jetty. In my case:

    $ cp target/web_test-0.1.0-SNAPSHOT-standalone.war ../../jetty-distribution-8.1.4.v20120524/webapps/

Now, change directory to your Jetty install, and copy the file contexts/test.xml which comes default to a new file:

    $ cp contexts/test.xml contexts/web_test.xml

Open it up:

    $ <editor> contexts/web_test.xml

Find the line:

    <Set name="war"><SystemProperty name="jetty.home" default="."/>/webapps/test.war</Set>

And edit this to match the war file you just copied over to the webapps/ directory.  In my case:

    <Set name="war"><SystemProperty name="jetty.home" default="."/>/webapps/web_test-0.1.0-SNAPSHOT-standalone.war</Set>

Then, a bit further down, comment out or delete this line:

    <Set name="overrideDescriptor"><SystemProperty name="jetty.home" default="."/>/contexts/test.d/override-web.xml</Set>

(Compare to the contexts/web_test.xml file in this repo for changes if confused.)

That's it.  At this stage, we'll test that things are working by starting up Jetty (from root of Jetty app):

    $ java -jar start.jar

If you open up a browser and head to

    http://<your IP or hostname>:8080/web_test-0.1.0-SNAPSHOT-standalone/

It's a bit lengthy, but we don't care for now since we're going to put it all behind an Apache proxy.

Note, everything beyond this point is basically Apache and Linux configuration stuff.  If getting a Clojure app running on Jetty is enough for you, you may want ot stop here.

## Setting up Jetty with Apache HTTPD

Now, we want our Clojure app to be there when you go to our domain. So we'll set up Apache as a proxy for our Jetty web app.  This more or less follows the "Configuring mod_proxy" section from the Jetty documentation linked to below.

To start with, add these two lines to your apache config.  For me this was in /etc/apache2/httpd.conf, at the bottom of a long list of other LoadModule calls:

    LoadModule proxy_module modules/mod_proxy.so
    LoadModule proxy_http_module modules/mod_proxy_http.so  # http://ubuntuforums.org/showthread.php?t=1434872

Then, add these lines to your vhost (or wherever your domain is set up...I'll assume you have that covered since this isn't an Apache tutorial):

    ProxyRequests Off
    ProxyVia Off
    ProxyPreserveHost On

    <Proxy *>
      AddDefaultCharset off
      Order deny,allow
      Allow from all
    </Proxy>

    ProxyPass / http://127.0.0.1:8080/web_test2-0.1.0-SNAPSHOT-standalone/

The Jetty Apache docs include the following lines, but I couldn't get it working, so I commented them out. YMMV.

    LoadModule status_module modules/mod_status.so  # (I assume you need this but it didn't help me.)

    ProxyStatus On
    <Location /status>
      SetHandler server-status
      Order Deny,Allow
      Allow from all
    </Location>

At this point, you should be ready to go.  Make sure Jetty is running, and restart Apache.  Go to your domain, and you should see your "Hello World" again.  Voila.

## Extra: getting Jetty running via a init script. 

However, you may not want to leave Jetty running on the command line forever, but would prefer to use a script to start it up and shut it down. In my case, I'm using Gentoo Linux, so I wanted something nice in my /etc/init.d folder to start and stop, and to configure as a default service, just like how Apache is set up (*TODO: configure for Gentoo rc-update so I can have this loaded by default*).

Now, for whatever reason Jetty isn't available as a package within Gentoo's emerge package management system on my server.  Luckily the Jetty package comes with some nice scripts available in the bin/ directory.  I just copied the one for *nix systems to my /etc/init.d directory:

    # from the root of the Jetty install directory
    $ cp bin/jetty.sh /etc/init.d/

The top of that file (bin/jetty.sh) provides a very detailed description of the options that are available for configuration including what files it tries to read from on startup.  It states that it reads from a file called /etc/default/jetty on initialization.  I also noted that by default the script tries to find the jetty install directory in /opt/jetty.  However, I preferred where I had it set up, so I added one variable to the file /etc/default/jetty:

    JETTY_HOME=/var/www/jetty-distribution-8.1.4.v20120524

Now, I can run

    $ /etc/init.d/jetty.sh start


## Documentation which was especially useful in writing this:

  * https://github.com/ring-clojure/ring/wiki/Getting-Started
  * http://www.enavigo.com/2008/08/29/deploying-a-web-application-to-jetty/
  * http://wiki.eclipse.org/Jetty/Tutorial/Apache
  * http://ubuntuforums.org/showthread.php?t=1434872 (didn't realize I needed mod_proxy_http!)

## TODO:

  * Get the init script configured with Gentoo's system so that I can have Jetty start up automatically on boot (http://www.gentoo.org/doc/en/handbook/handbook-x86.xml?part=2&chap=4#doc_chap2).

## License

Copyright &copy; 2012 Dave Della Costa

Distributed under the MIT License (http://dd.mit-license.org/)
