---
layout: post
title: "On Foundation's validation &amp; my Pull Request"
date: 2014-09-23 02:51:42 +0530
comments: true
categories: 
---

We widely use the amazing [Zurb Foundation](http://foundation.zurb.com/) framework as a base for many of our projects at work. Foundation provides a lot of features and utilities out of the box. One such useful feature is the JavaScript form validation library which they call [Abide](http://foundation.zurb.com/docs/components/abide.html).


Abide is powerful and also flexible. There are a lot of configuration options and it even allows us to easily add custom validations in addition to the ones it provides out of the box. Foundation Abide was working out fine for us until recently when a client of ours asked for a change in the way validation works in the app we do for them.
<!--more-->
By default Abide performs validation whenever the _change_ or _blur_ events (and _keyup_ if _live_validation_ is enabled) are fired. Our client wanted to disable this and wanted validation to happen only upon form submit. That sounded simple since I assumed there should be an option to turn it off. But when I checked the documentation to find the right option I found that there was none to disable validation on blur or change.

So then as always I did a Google search to see if other people have figured this one out and sure enough [someone else had asked](http://foundation.zurb.com/forum/posts/1543-disable-abide-form-validation-on-blur-the-fields) for the same thing on the Foundation forums. But the solution given was a simple hack to override foundations blur and change event callbacks with our own which does nothing.

{% codeblock validate_on_blur.js %}
$('input, textarea, select')
 .off('.abide')
 .on('blur.fndtn.abide change.fndtn.abide', function (e) {
  // do nothing
});
{% endcodeblock %}

And that worked perfectly and we shipped the changes making our client and satisfied. But I was not so happy since, as you might think, this is not the right way to do this and it felt kinda dirty to solve it this way. And so I began looking into the [Abide library's code](https://github.com/zurb/foundation/blob/master/js/foundation/foundation.abide.js) in the Foundation GitHub repository hoping to find some undocumented way to get this done but found none.

It's been a really long while since I last contributed something to an open source project and so I thought why not implement this feature myself and send a pull request to Foundation. That would make me happy since I love contributing and it would also help me make my project better! Just to find out what the maintainers thought about this I opened an [issue](https://github.com/zurb/foundation/issues/5760) and they were happy with the new feature and asked for the code.

Coding this feature was pretty straightforward since I just had follow the existing patterns in the library to add this new option which I named 'validate_on_blur'. Once I implemented the feature I updated the docs so this new option is reflected there and ran the Jasmine tests with Karma to make sure nothing broke. I then kinda felt compelled to write a spec for the new option and so I wrote a spec and made sure it passed.

I've sent a [pull request](https://github.com/zurb/foundation/pull/5774) on GitHub with this code and I'm waiting for the maintainers to merge it in. I will update this post once I get a response. I strongly feel this is one of the best ways for people to contribute to their favorite open source projects and I am happy to have been able to make this contribution. Thanks to our client who requested this feature in the first place ;)