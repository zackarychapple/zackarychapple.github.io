---
layout: post
title:  "Angular2 Wysiwig"
date:   2016-12-01 11:40:56 -0500
categories: angular2 tinymce
---
## Intro

WSYWIG editors are sometimes a necessity in our development lives. Using [TinyMce][tiny] is actually very easy.

In order to start using TinyMce we need to install the dependency. There is an npm module for tinymce.

### package.json

{% highlight javascript %}
"tinymce": "^4.5.0",
{% endhighlight %}

We have to then ad the tinymce javascript files to the bundler for the angular-cli.

### angular-cli.json

{% highlight javascript %}
  "../node_modules/tinymce/tinymce.js",
  "../node_modules/tinymce/themes/modern/theme.js",
  "../node_modules/tinymce/plugins/link/plugin.js",
  "../node_modules/tinymce/plugins/paste/plugin.js",
  "../node_modules/tinymce/plugins/table/plugin.js"
{% endhighlight %}


Creating the component is actually pretty straight forward. The two key components to pay attention to is the after view init and the destroy.
You need to make sure that you initialize the tinymce editor and tear it down.

### wyswig.component.ts

{% highlight javascript %}
import { Component, AfterViewInit, OnDestroy } from '@angular/core';
import { Input, Output } from '@angular/core/src/metadata/directives';
import { EventEmitter } from '@angular/forms/src/facade/async';

declare let tinymce: any;

@Component({
  selector: 'wysiwyg-editor',
  templateUrl: './wysiwyg-editor.component.html',
  styleUrls: [ './wysiwyg-editor.component.css' ]
})
export class WysiwygEditorComponent implements AfterViewInit, OnDestroy {
  editorId: string;

  constructor() {
    this.editorId = `editor-${Math.floor(Math.random() * 10000)}`;
  }

  ngAfterViewInit() {
    let options = {
      skin_url: 'assets/skins/lightgray',
      selector: '#' + this.editorId,
      setup: (editor: any) => {
        this.editor = editor;

        this.editor.on('change', () => {
          const newContent = editor.getContent();
          this.editorChange.emit(newContent);
        });

        this.getEditor.emit(this.editor);
      }
    };
    Object.assign(options, this.editorOptions);
    tinymce.init(options);
  }

  ngOnDestroy(): void {
    tinymce.remove(this.editor);
  }

}
{% endhighlight %}

### editor.component.html

{% highlight javascript %}
<textarea [id]="editorId"{ {dom_content} }</textarea>
{% endhighlight %}

### theme

Copy theme assets from node_modules to the assets directory.

## Display

### Display HTML

{% highlight html %}
<div #dom></div>
{% endhighlight %}

### View Component
The component itself can hook into the elementRef in our display HTML file.
{% highlight javascript %}
@ViewChild('dom') dom: ElementRef;
{% endhighlight %}


Then we can set the innerHTML of our nativeElement to the content from our wyswig element.
{% highlight javascript %}
this.dom.nativeElement.innerHTML = `${data}`;
{% endhighlight %}

## NativeScript

If you want to show the dom from the wysiwig editor you can use a WebView tag and put the dom as the src for the tag.  Make sure that you set a height attribute if have the element in a StackLayout as without it the WebView item has no height.

{% highlight html %}
<WebView  height="1500" [src]="dom"></WebView>
{% endhighlight %}

[tiny]: https://www.tinymce.com/
[wysiwig]: https://github.com/zackarychapple/ng2-tinymce-wyswig
