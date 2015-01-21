---
layout: post
title: ""Ruby 2.0 boosts Rails boot time""
date: 2015-01-21 19:23:54 +0530
comments: true
categories: 
published: false
---

Notes:

http://rubylearning.com/satishtalim/mutable_and_immutable_objects.html
https://bugs.ruby-lang.org/issues/7158
http://magazine.rubyist.net/?Ruby200SpecialEn-require

### Old Snapshot app:
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

### New App:

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
