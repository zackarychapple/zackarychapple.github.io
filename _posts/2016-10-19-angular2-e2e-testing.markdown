---
layout: post
title:  "Angular2 E2E Testing with Tour of Heroes using Async Await"
date:   2016-10-19 08:40:56 -0500
categories: angular2
---
## Intro

With the Angular-CLI it is incredibly easy to create an application, however one thing that I have been struggling to
find documentation on is e2e testing.

It seems the simplest way to begin exploring e2e testing is to start with [John Papa’s][john] [Tour of Heroes][toh] repo.
After recreating the application with the CLI it was time to do some basic tests, or so I thought.

As I started to create the tests I realized that I was getting into promise hell. I’m sure you’ve heard of callback hell,
but this is worse, about three layers deep of promise then layers I started thinking that there must be a better way.

Recently I had the opportunity to get some assistance on a node project from [Blake Embrey][blake] and in the process of
the refactor of that node server he started showing me how to use async await.  Admittedly I am not an expert but this sparked some ideas.
Could I use this same methodology to help simplify my test stack?  Turns out I could.

## Code

### tsconfig.json

In order to use async await you will need to update your compilation target in your `tsconfig.json`.

{% highlight javascript %}
{
  "compileOnSave": false,
  "compilerOptions": {
    "declaration": false,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "module": "commonjs",
    "moduleResolution": "node",
    "outDir": "../dist/out-tsc-e2e",
    "sourceMap": true,
    "target": "es6",
    "typeRoots": [
      "../node_modules/@types"
    ]
  }
}
{% endhighlight %}

### package.json

I also added the protractor and selenium-webdriver types to my project.  Note the specific version for selenium-webdriver.
I ran into some issues when using newer versions and after some github-fu I found that this version works.

{% highlight javascript %}
"@types/protractor": "^1.5.20",
"@types/selenium-webdriver": "2.44.29",
{% endhighlight %}


### app.po.ts

The `app.po.ts` file is where you should be putting functionality that is shared across your e2e tests, at least that is how
I understood the initial scaffold.  Because we want to not have to deal with promises you can add async to each of the
function blocks. After you do that you can add the await to the return and this will allow you to write code synchronously.
When starting to look into async await I found [this article][ponyfoo] to be the most helpful.

{% highlight javascript %}
export class E2eDemoPage {

  async navigateTo() {
    return await browser.get('/');
  }

  async getParagraphText() {
    return await element(by.css('my-app h3')).getText();
  }

  async getElementText(selector: string) {
    return await element(by.css(selector)).getText();
  }

  async getHero(id: string) {
    await browser.driver.findElements(by.id(id));
    return await element(by.id(id)).click();
  }

}
{% endhighlight %}

### app.e2e-spec.ts

This is actually where the biggest gotcha's were.  I was fighting for a long time to try and get async await to work
and it wasn't until Blake suggested that I add async to the `it` blocks themselves that everything started to come together.

{% highlight javascript %}
import { E2eDemoPage } from './app.po';

describe('e2e-demo App', ()=> {
  let page: E2eDemoPage;

  beforeEach(async() => {
    page = new E2eDemoPage();
    await page.navigateTo();
  });

  it('should display message saying Top Heroes', async() => {
    expect(page.getParagraphText()).toEqual('Top Heroes');
  });

  describe('navigation events', async()=> {
    const hero = 'Narco';
    const heroSelector = 'my-hero-detail h2';
    const backSelector = 'back';
    beforeEach(async() => {
      await page.getHero(hero);
    });
    it('to hero when clicked', async() => {
      await browser.driver.findElements(by.css(heroSelector));
      const elementText = await page.getElementText(heroSelector);
      expect(elementText).toBe(`${hero} details!`);
    });

    it('should navigate back to dashboard when back is clicked', async()=> {
      await browser.driver.findElements(by.id(backSelector));
      await element(by.id(backSelector)).click();
      expect(page.getParagraphText()).toEqual('Top Heroes');
    })
  })

});

{% endhighlight %}

## Assumptions
I know these tests are far from perfect but I hope it helps provide a start for anyone looking to explore e2e testing and
the angular cli.

## Repo
If you want to see the full repo you can find it [here][repo] if you have any questions or comments please feel free to open
a PR or create an issue.

## Thanks
This post is only possible thanks to [Blake Embrey][blake] and [John Papa][john] and the [Angular CLI team][cli]

[ponyfoo]: https://ponyfoo.com/articles/understanding-javascript-async-await
[toh]: https://github.com/johnpapa/angular2-tour-of-heroes
[repo]: https://github.com/zackarychapple/ng2-testing-demo
[blake]: https://twitter.com/blakeembrey
[john]: https://twitter.com/John_Papa
[cli]: https://github.com/angular/angular-cli/graphs/contributors