---
layout: post
title: "Hosting FreeGeoIP in your cloud"
date: 2014-11-30 21:11:02 +0530
comments: true
categories: ruby go devops
---

In the project I'm working on where we sell photos, we had to find out the country of each and every visitor for a certain business purpose. So we have this piece of code in the Rails `ApplicationController` to do the job.

``` ruby
def set_country_currency
  begin 
    session[:country] ||= request.location.country
  rescue
    logger.error "ip=#{request.remote_ip} exception_type=visitor_geocoding_failure"
    session[:country] = "United States of America"
  end
end
```
The `location` method called on `request` is actually provided by this super awesome and popular Geocoding gem called [geocoder](https://github.com/alexreisner/geocoder) and it returns the country from which the request originated. It figures this information out using the IP address available in the request object. Okay but how does the geocoder gem finds the country from an IP address? For doing that geocoder actually talks to an external service called [FreeGeoIP](http://freegeoip.net) which is a service written in Go-lang. Btw, Geocoder supports many more such geocoding services like the ones provided by Google, MaxMind etc. FreeGeoIP is completely free and has a fairly reasonable rate-limit (10000 requests / hr). We deployed this part to production and it was working fine until I noticed some performance issues within couple of days.

As most of you might have guessed, making a request to an external service can be a pretty costly affair and can ruin the response times of your app if you are using it in a crucial place like how we use in our app. The worst part is that whenever a new user visits our site for the first time they are forced to go through this process and consequently the inital page load took too long. Below is the breakdown table for our landing page's controller action provided by NewRelic. As you can see this request to freegeoip.net is slowing things down a lot.

{% img http://i.imgur.com/MwAoGBc.png %}

The good thing about FreeGeoIP is the fact that its an [OpenSource](https://github.com/fiorix/freegeoip) app and can be hosted by anyone anywere. Since our app is on Amazon AWS, a simple fix to this latency issue would be to host FreeGeoIP inside our cloud so that the latency becomes almost negligible. But I never had the urgency to do this as we had other tasks on our plate which I felt were more important and postponed this task to be done at the end of the week.

But then before the week could end disaster struck. I woke up and found a string of emails on my phone alerting me about the error rate being too high on the app. I logged onto Rollbar (a service we use for exception tracking) and found the exception - `Errno::ECONNREFUSED: Connection refused - connect(2)` thrown everywhere we used the geocoding service. I initially thought that we might have hit the rate limit or something but just to make sure I tried visiting freegeoip.net and found that the site was unreachable. That was it. This was the worst case scenario that could have happened and it happened and I had to take some action. 

{% img http://i.imgur.com/stSG9rp.png %}

At first I tried switching to MaxMind as a geocoding service but the format in which the results were sent back was different from that received from FreeGeoIP and I would have to make some changes in my app to accomodate this and I did not feel good about that. So the only choice that remained was to host FreeGeoIP on our own in our cloud. 

Although this sounds simple, the main problem was that FreeGeoIP is a Go app and I've never worked with a Go project before. The documentation was also not very helpful (planning to send patch for it). But in the end I managed to find my way through with some help from the steps inside a [Dockerfile](http://www.spritle.com/blogs/2013/08/23/docker-for-beginners/) in the repo :)

So here's how you build the FreeGeoIP project (or any Go project).

#### 1. Install Go

For Debian based Linux systems installing is pretty straightforward. All you need to do to run these commands, 

``` bash
$ sudo apt-get install python-software-properties  # 12.04
$ sudo add-apt-repository ppa:duh/golang
$ sudo apt-get update
$ sudo apt-get install golang
```

[Pravin Mishra](http://railskey.wordpress.com/2014/05/31/install-gogolang-on-ubuntu/) has written a nice blog detailing all this. Also you can find more instructions in the official [Go-lang website](http://golang.org/doc/install).

Once done you need to setup two environment variables. One is `GOROOT` which points to the Go installation (`/usr/lib/go`, if you used above commands to install). The other is the `GOBIN` variable which points to the directory where you want to keep the binary files generated after building your Go projects. (say `/usr/bin/g`).

Once done type `go version` on your terminal to confirm the installation.

#### 2. Setup your workspace

Go requires your project directory to be setup in a certain way. So lets get that setup. 

First lets create a root directory for our Go projects.

``` bash
$ mkdir ~/go-lang
```

Now under this directory you need to have this directory called `src` which will contain the Go source files. It is important that your source files are stored under a directory hierarchy that follows the source control repository URLs of the Go projects. 

For instance the FreeGeoIP project's source code needs to be under the directory - `~/go-lang/src/github.com/fiorix/freegeoip`. This way projects are automatically namespaced. Pretty neat. Let's go ahead and set this up.

``` bash
$ mkdir -p ~/go-lang/src/github.com/fiorix/freegeoip
$ cd ~/go-lang/src/github.com/fiorix/freegeoip
$ git clone https://github.com/fiorix/freegeoip.git . 
``` 
Now we've got the workspace setup with the source code of the FreeGeoIP app. Let's build it.

#### 3. Build the app

The source code for the web service is under the `cmd/freegeoip` directory. So navigate into it and run,
``` bash
$ go get
```
This dowloads all the dependent packages. This is something similar to `bundle install` in Ruby. Now run,
``` bash
$ go install
``` 
This builds the Go app and puts the binary file into the `GOBIN` directory. In our case its `/usr/bin/g`.
``` bash
$ ls /usr/bin/g
freegeoip
``` 
#### 4. Start the server

To run the app simply navigtate to the `/usr/bin/g` folder and run
``` bash
$ ./freegeoip
```
This will boot up the freegeoip server which listens by default at port 8080. At first boot it downloads the IP database from MaxMind. Once its done you can start sending requests to it. Try this:
``` bash
$ curl -i http://localhost:8080/json/8.8.8.8
{"ip":"8.8.8.8","country_code":"US","country_name":"United States","region_code":"CA","region_name":"California","city":"Mountain View","zip_code":"","time_zone":"America/Los_Angeles","latitude":37.386,"longitude":-122.084,"metro_code":807}
```
This returns the geolocation details of the IP address `8.8.8.8` in JSON format. Awesome right? 

#### 5. Install as a service

Now this server is attached to the terminal. Once you terminate the terminal or disconnect SSH the server will be terminated. 

Let's install the app as a service in our server using Ubuntu [Upstart](http://upstart.ubuntu.com/) so that it can run as a daemon and can be managed easily. I've never done this kinda thing before but doing this was fairly straightforward. All you have to do is drop a configuration file into the `/etc/init` directory and BOOM you can do things like `start service_name`, `stop service_name`, `status service_name`, etc to manage the service. Going through Upstart's docs was a pretty good experience and I found this wonderful [article](http://stackful-dev.com/what-every-developer-needs-to-know-about-ubuntu-upstart.html) along the way as well. I also found out that Upstart was going to be replaced by something called `systemd`. 

Anyways, the FreeGeoIP project comes with an upstart conf file. But I wanted a little more. I wanted it to print out its PID to a file so that the service can be tracked and monitored (we'll come to the bit about how its done later). So here is the conf file I used.

``` bash freegeoip.conf
description "freegeoip web server"

start on runlevel [2345]
stop on runlevel [!2345]

script
  echo $$ > /var/run/freegeoip.pid
  exec /usr/local/freegeoip -silent
end script

post-start script
   echo "freegeoip started"
end script
```

I am not gonna explain this thing line by line but its all incredibly easy to understand if you refer the Upstart documentation. Once you got this in place (`/etc/init/freegeoip.conf`), run the following command to refresh Upstart. People claim this step is not needed but I had to do this several times to get Upstart to recognise the new configuration file.

``` bash
$ initctl reload-configuration
```

If everything is fine, you should see `freegeoip` in the list given by the command,

``` bash
$ initctl list
```

Now you can simply do,

``` bash
$ start freegeoip
```

to startup the service. Try doing `status freegeoip` to make sure its running. You should also see that the PID of the freegeoip process is found in the `/var/run/freegeoip.pid` file. At this point everything is setup. You can go ahead and configure the geocoder gem to use this own FreeGeoIP installation for geocoding. But since this is a critical service for our app I wanted to add some safety measures in place.

#### 6. Configuring with Monit

We use [Monit](http://mmonit.com/monit/) which is an extremely lightweight system monitoring and error recovery tool that can watch the processes or files you want and take actions when certain things happen like restarting your app when it goes down, restarting your processes if they take more memory, etc and also it allows you to setup alerts. Monit also provides a nice web interface using which you can get a glimpse of the processes running in your server. Installing Monit is super easy. Just follow the instructions in the website.

Once you've got Monit installed, configuring it is pretty simple. Here is a simple configuration we can use for managing the `freegeoip` process.

```
check process freegeoip with pidfile /var/run/freegeoip.pid
  start program = "/sbin/start freegeoip"
  stop program = "/sbin/stop freegeoip"
```

Very expressive. Right? We tell monit to keep track of the process with name `freegeoip` with the PID form the file `/var/run/freegeoip.pid` (now do you get why I wanted the PID so badly?). We then tell monit how to start and stop the process. By default monit will keep watch and when the process goes down for some reason monit will start it back up again. Thus there would be minimal disruption in service.

Add this configuration into a file `monit-freegeoip.conf` in the `/etc/monit/monit.d` directory. By default whatever file you add here will be included in the `/etc/monit/monitrc` file which is the main configuration file. This happens due to this line at the bottom of the `monitrc` file - `include /etc/monit/monit.d/*.conf`.

Restart monit using `sudo service monit restart` and do `monit status`. You'll find info about the `freegeoip` process in the output. It should be something like this - 

```
Process 'freegeoip'
  status                            Running
  monitoring status                 Monitored
  pid                               12155
  parent pid                        1
  uptime                            4d 6h 57m 
  children                          0
  memory kilobytes                  7084
  memory kilobytes total            7084
  memory percent                    0.4%
  memory percent total              0.4%
  cpu percent                       0.0%
  cpu percent total                 0.0%
  data collected                    Mon, 01 Dec 2014 07:54:17
```

Great. Now if freegeoip goes down for some reason, monit will start it right back up again and we can sleep peacefully without any worries :)

Lets do a small recap of what we've done so far.

1. We installed Go
2. We built and ran the FreeGeoIP Go project
3. We installed it as a service using Upstart
4. We configured Monit to monitor the process

Hope this was useful and not boring :)

Back to my problem. Once I got our very own FreeGeoIP service running, all I had to do was configure the Geocoder gem to use this instead of freegeoip.net. Upon consulting the documentation I learned that it was as easy as doing the following in a Rails initializer.

``` ruby
Geocoder.configure(
  timeout: 10,
  :ip_lookup => :freegeoip,
  :freegeoip => {
    :host => "myserver.com:8080"
  }
)
```
And now for the moment of truth. I hooked onto Rails console and typed the following statement - `Geocoder.search "8.8.8.8"` and what do I get? `Errno::ECONNREFUSED: Connection refused - connect(2)`

Same old error again. I was completely puzzled. I checked monit to see if the service was still up and it was up. I used `curl` again to hit the freegeoip service and it was working fine. Something somewhere was going horribly wrong. I realized something was up with the geocoder gem and so I opened up the source code on GitHub to see where the `:host` configuration was being used and try to find out if it was really using the host supplied by me.

This method in the file [`lib/geocoder/lookups/freegeoip.rb`](https://github.com/alexreisner/geocoder/blob/master/lib/geocoder/lookups/freegeoip.rb#L11) was responsible for building the query URL and it used this `host` private method which inturn checked the `configuration` hash to see if `:host` was supplied and if so it used it and if not it fellback to `freegeoip.net`.

``` ruby freegeoip.rb
def query_url(query)
  "#{protocol}://#{host}/json/#{query.sanitized_text}"
end

private

def host
  configuration[:host] || "freegeoip.net"
end
```

All seems fine. I decided to log this `query_url` to the console to see what was being built. Now I had to edit the gem's source on my machine. To do this I used `bundle open` to open up the geocoder source code locally. When I opened up the `freegeoip.rb` file, what I found was slightly different from what I saw on GitHub. Here's the `query_url` method that I had on my local development setup.

``` ruby
def query_url(query)
  "#{protocol}://freegeoip.net/json/#{query.sanitized_text}"
end
```

As you can see there was no way to configure the host here. As it turned out I was using an older version of the gem. So I went ahead and did a `bundle update geocoder` and then ran `Geocoder.search "8.8.8.8"` and I got,

``` ruby
=> [#<Geocoder::Result::Google:0x000000062888a8
  @cache_hit=nil,
  @data=
   {"address_components"=>
     [{"long_name"=>"Route 8", "short_name"=>"LA-8", "types"=>["route"]},
      {"long_name"=>"Louisiana", "short_name"=>"LA", "types"=>["administrative_area_level_1", "political"]},
      {"long_name"=>"United States", "short_name"=>"US", "types"=>["country", "political"]}],
    "formatted_address"=>"Louisiana 8, Louisiana, USA",
    "geometry"=>
     {"bounds"=>{"northeast"=>{"lat"=>31.8475579, "lng"=>-91.6569889}, "southwest"=>{"lat"=>31.0643337, "lng"=>-93.5195453}},
      "location"=>{"lat"=>31.523812, "lng"=>-92.587424},
      "location_type"=>"GEOMETRIC_CENTER",
      "viewport"=>{"northeast"=>{"lat"=>31.8475579, "lng"=>-91.6569889}, "southwest"=>{"lat"=>31.0643337, "lng"=>-93.5195453}}},
    "partial_match"=>true,
    "types"=>["route"]}>]
```

Massive relief. Finally!

Deployed all this to production and within some hours we noticed improvements in the response times. And the breakdown table from NewRelic for the same landing page action now shows how much we have improved. The request to the geocoding service is down below in the table.

{% img http://i.imgur.com/ueFn1cz.png?1 %}

For me this was an extremely good learning experience. I hope this blog was useful and interesting for you as well :)