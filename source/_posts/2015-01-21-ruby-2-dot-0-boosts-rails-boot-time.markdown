---
layout: post
title: Ruby 2.0 boosts Rails boot time
date: 2015-01-21 19:23:54 +0530
comments: true
categories: ruby
published: true
---

I've been working on a codebase which is Rails 4.0 and Ruby 1.9.3 and something that's annoying about this app is its incredibly large load time. It takes about 26 seconds to boot up.

`bundle exec rake environment  22.22s user 1.14s system 91% cpu 25.632 total`

Now we're not doing anything fancy during the bootup so I figured it must the gems that we are loading that's taking so much time. 
While googling for ways to debug the boot time I came across this really cool gem called [Bumbler](https://github.com/nevir/Bumbler) which prints out the load times for each and every gem in your app.

These are some of the gems that were taking lots of time to load.

```
    367.43  rails
    408.94  fuubar
    420.56  slim
    441.58  pry-rails
    488.60  activemerchant
    651.05  meta-tags
    986.72  axlsx
    1231.87  newrelic_rpm
    1303.99  geocoder
    2749.84  fog
    3044.00  carrierwave
```

Its amazing that it takes 3 seconds to load carrierwave. 
I double checked my initializers to make sure there's nothing that could slow down the loading and there was none. 
So is Ruby's `require` slow?

To find out I did some digging and ended up at this [bug report](https://bugs.ruby-lang.org/issues/7158) which talks about improvements to Ruby's `require` method.
It seems when requiring a file Ruby iterates through the `$LOAD_PATHS` array to see if the file is loaded which is extremely ineffecient if there are many files to load. 
The patch made uses a hash to maintain the loaded files so that it can be looked up in O(1) time.
This is explained by Xavier Shay [here](http://rhnh.net/2011/05/28/speeding-up-rails-startup-time).

In addition to this the string objects in the arrays `$LOAD_PATHS` and `$LOADED_FEATURES` are also frozen so these can be cached. 
Freezing strings makes them immutable. This is well explained in this [other article](http://magazine.rubyist.net/?Ruby200SpecialEn-require) by [Masaya Tarui](https://twitter.com/taru), author of the patch.

All this has made Ruby 2.0 require files much faster. I temporarily switched to Ruby 2.2 and take a look at this - 
`bundle exec rake environment  3.95s user 0.85s system 99% cpu 4.822 total`.

Bumbler report: 

```
    102.74  countries
    103.72  haml-rails
    117.40  devise
    122.33  foundation-rails
    130.76  newrelic_rpm
    132.80  pry-rails
    144.39  sass-rails
    163.59  activemerchant
    169.82  rails
    195.35  axlsx
    261.47  geocoder
    632.77  fog
    780.72  carrierwave
```



That's a whopping 81% improvement over Ruby 1.9.3 in my case. Your mileage may vary. But this is amazing. So as soon as I get the chance I am going to upgrade to Ruby 2.2 :)