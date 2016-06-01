---
layout: post
title:  "ng2-translate on AngularCLI.Beta5"
date:   2016-05-31 21:34:56 -0400
categories: angular2
---
## Intro
I needed i8tn in a project I'm working on.  [Olivier Combe][olivier] created a wonderful implementation of angular translate in angular2, [ng2-translate][translate]. 

## Baseline
The directions are pretty plug and play but since I use the [angular-cli][cli] there are some additional complexities

## Setup
* The first step is performing an install `npm install ng2-translate --save` this part is pretty straight forward. 
* Next we need to begin editing a couple of AngularCLI files in order to help the ng2-translate modules get to the dist folder and be consumable. 
    1. We need to add `'ng2-translate/**/*.+(js|js.map)',` to our `angular-cli-build.js`.  This makes it so the CLI picks up the files as part of the build process and moves them to the dist directory.
    2. In our `src/system-config.ts` we need to add `'ng2-translate',` to the third party barrels section and `'ng2-translate':'vendor/ng2-translate',` to the map section. These two changes make it so systemJS will pick up our files and where to find them when we import them.
* Lets create our first translation file `en.json` and put this in `src/assets/i8tn`, this is in the guide Olivier put on github as an example.
{% highlight json %}
{
    "HELLO": "hello {{value}}"
}
{% endhighlight %}
* In the root component of our app we need to add a few imports.
{% highlight javascript %}
import { Component, OnInit, provide } from '@angular/core';
import { TranslateLoader, TranslateService, TranslateStaticLoader } from "ng2-translate/ng2-translate";
import { Http } from '@angular/http'
{% endhighlight %}
* Also in the root component we also need to add the translation providers per the instructions. 
{% highlight javascript %}
provide(TranslateLoader, {
  useFactory: (http: Http) => new TranslateStaticLoader(http, '/assets/i8tn', '.json'),
  deps: [Http]
}),
TranslateService, SearchHistoryService 
{% endhighlight %}
* In the constructor of our root component we can also set the default and current language. 
{% highlight javascript %}
constructor(private translate:TranslateService,){
    let userLang = navigator.language.split('-')[0]; // use navigator lang if available
    userLang = /(fr|en)/gi.test(userLang) ? userLang : 'en';
    translate.setDefaultLang('en');
    translate.use(userLang);
}
{% endhighlight %}
* In each component with text content we want to translate we need to add the translation pipe in the metadata for our component `pipes: [TranslatePipe]`
* With the pipe we can begin to use the translations the pipe will match the keys in the json file, for example `HELLO` is the key being used. The `param` is an attribute value on the component class itself.
{% highlight html %}
 {% raw %}
<div>{{ 'HELLO' | translate:{value: param} }}</div>
 {% endraw %}
{% endhighlight %}

[translate]: https://github.com/ocombe/ng2-translate
[olivier]: https://twitter.com/ocombe
[cli]: https://github.com/angular/angular-cli