---
layout: post
title:  "Angular2 Testing with Tour of Heroes"
date:   2016-11-25 08:40:56 -0500
categories: angular2
---
## Intro

Testing is something near and dear to my heart. When we launched our first production Angular2 application into the wild two weeks ago
the question was asked, "Where are the tests?".  We had a few [e2e tests][e2e] that covered core functionality but now that we
have a chance to look over our last 8 months of work we realized we need to refactor quite a bit of the code.

Before refactor though we needed to wrap our code in unit tests as well.

## Testing Services (Easy Example)

One of the easiest things for us to test are simple http services.  Tour of Heroes had a good example of a simple http service.

### hero-search.service.ts

{% highlight javascript %}
@Injectable()
export class HeroSearchService {
  constructor(private http: Http) {
  }

  search(term: string): Observable<Hero[]> {
    return this.http
      .get(`app/heroes/?name=${term}`)
      .map((r: Response) => r.json().data as Hero[]);
  }
}
{% endhighlight %}

### hero-search.service.spec.ts

In order to begin our unit testing we have to configure our `TestBed`. Angular2 provides a testing module that we can configure and control
 our injections.  I found that having the configuration in a `beforeEach` block allows for easy setup and tear-downs.

For this example we need to setup http.

{% highlight javascript %}
beforeEach(() => {
  TestBed.configureTestingModule({
    providers: [
      HeroSearchService,
      MockBackend,
      BaseRequestOptions,
      {
        provide: Http,
        useFactory: (backend: MockBackend, options: BaseRequestOptions) => new Http(backend, options),
        deps: [ MockBackend, BaseRequestOptions ]
      }
    ]
  });
});
{% endhighlight %}

After our test bed is configured we need to actually inject http into the service itself.

{% highlight javascript %}
beforeEach(inject([ MockBackend, Http ],
    (mb: MockBackend, http: Http) => {
        mockBackend = mb;
        heroSearchService = new HeroSearchService(http);
}));
{% endhighlight %}

With the setup complete we can actually test our service. Because this is going to be asynchronous we need to have a callback for Jasmine to know when
 the test is done executing.

 While the service itself is simple we can test a few things on it that might come in handy later. By hooking into the `mockBackend`'s connections
 we can verify the method, url, and mock a response. This allows us to verify the correct url was called, with the correct parameter.
 Because our `search` function is returning an observable we can also verify any side effects we may be using on the response. In this case there are none
 but we still verify the data coming back.

{% highlight javascript %}
it('should return observable with hero array', (done) => {
  let searchTerm = 'some term';
  mockBackend.connections.subscribe((connection: MockConnection) => {
    expect(connection.request.method).toEqual(RequestMethod.Get);
    expect(connection.request.url).toEqual('app/heroes/?name=some term');
    connection.mockRespond(new Response(new ResponseOptions({
      body: {data: MockHeroesArray}
    })))
  });
  heroSearchService.search(searchTerm).subscribe(result => {
    expect(result).toEqual(MockHeroesArray);
    done();
  })
})
{% endhighlight %}

Here is the test as a whole for our `hero-search.service`.

{% highlight javascript %}
describe('Service: HeroSearch', () => {
  let mockBackend: MockBackend;
  let heroSearchService: HeroSearchService;
  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        HeroSearchService,
        MockBackend,
        BaseRequestOptions,
        {
          provide: Http,
          useFactory: (backend: MockBackend, options: BaseRequestOptions) => new Http(backend, options),
          deps: [ MockBackend, BaseRequestOptions ]
        }
      ]
    });
  });

  beforeEach(inject([ MockBackend, Http ],
    (mb: MockBackend, http: Http) => {
      mockBackend = mb;
      heroSearchService = new HeroSearchService(http);
    }));
  it('should ...', inject([ HeroSearchService ], (service: HeroSearchService) => {
    expect(service).toBeTruthy();
  }));

  it('should return observable with hero array', (done) => {
    let searchTerm = 'some term';
    mockBackend.connections.subscribe((connection: MockConnection) => {
      expect(connection.request.method).toEqual(RequestMethod.Get);
      expect(connection.request.url).toEqual('app/heroes/?name=some term');
      connection.mockRespond(new Response(new ResponseOptions({
        body: {data: MockHeroesArray}
      })))
    });
    heroSearchService.search(searchTerm).subscribe(result => {
      expect(result).toEqual(MockHeroesArray);
      done();
    })
  })
});
{% endhighlight %}


## Testing Services (Through Example)
For a much more verbose example we have the `hero.service` which converts the observables to promises, and also has some catch logic.

### hero.service.ts

{% highlight javascript %}
@Injectable()
export class HeroService {
  heroesUrl = 'app/heroes';

  constructor(private http: Http) {
  }

  getHeroes(): Promise<Hero[]> {
    return this.http
      .get(this.heroesUrl)
      .toPromise()
      .then(response => response.json().data as Hero[])
      .catch(this.handleError);
  }

  getHero(id: number): Promise<Hero> {
    return this.getHeroes()
      .then(heroes => heroes.find(hero => hero.id === id));
  }

  save(hero: Hero): Promise<Hero> {
    if (hero.id) {
      return this.put(hero);
    }
    return this.post(hero);
  }

  delete(hero: Hero): Promise<Response> {
    let headers = new Headers();
    headers.append('Content-Type', 'application/json');

    let url = `${this.heroesUrl}/${hero.id}`;

    return this.http
      .delete(url, {headers: headers})
      .toPromise()
      .catch(this.handleError);
  }

  // Add new Hero
  private post(hero: Hero): Promise<Hero> {
    let headers = new Headers({
      'Content-Type': 'application/json'
    });

    return this.http
      .post(this.heroesUrl, JSON.stringify(hero), {headers: headers})
      .toPromise()
      .then(res => res.json().data)
      .catch(this.handleError);
  }

  // Update existing Hero
  private put(hero: Hero): Promise<Hero> {
    let headers = new Headers();
    headers.append('Content-Type', 'application/json');

    let url = `${this.heroesUrl}/${hero.id}`;

    return this.http
      .put(url, JSON.stringify(hero), {headers: headers})
      .toPromise()
      .then(() => hero)
      .catch(this.handleError);
  }

  handleError(error: any): Promise<any> {
    console.error('An error occurred', error);
    return Promise.reject(error.message || error);
  }
}
{% endhighlight %}

### hero.service.spec.ts

Since this test has a little more complexity we need to have some differences for the test bed configuration. Instead of having it in the before each block
I decided to move it to a function block. I'm going to pass into this block a failed or successful http connection.

For the failed request we can create a simple class.

{% highlight javascript %}
class MockFailedGetHeroesHttp extends Http {
  constructor(backend, options) {
    super(backend, options)
  }

  get() {
    return Observable.throw('error');
  }
}
{% endhighlight %}

The same is true for the successful request.
{% highlight javascript %}
class MockSuccessGetHeroesHttp extends Http {
  constructor(backend, options) {
    super(backend, options)
  }

  get() {
    return Observable.from([ new Response(new ResponseOptions({body: {data: MockHeroesArray}})) ]);
  }
}
{% endhighlight %}

The setup function itself is very similar to the setup we used for the simple service except the httpMock is passed in.
{% highlight javascript %}
let setup = function (httpMock) {
  TestBed.configureTestingModule({
    providers: [
      HeroService,
      MockBackend,
      BaseRequestOptions,
      {
        provide: Http,
        useFactory: (backend: MockBackend, options: BaseRequestOptions) => new httpMock(backend, options),
        deps: [ MockBackend, BaseRequestOptions ]
      }
    ]
  });
  inject([ MockBackend, Http ],
    (mb: MockBackend, http: Http) => {
      mockBackend = mb;
      heroService = new HeroService(http);
    })();
};
{% endhighlight %}

Here is the complete test for the hero service.
{% highlight javascript %}
describe('Service: Hero', () => {
  it('should call handle error from the promise when getHeroes fails', (done) => {
    setup(MockFailedGetHeroesHttp);
    spyOn(heroService, 'handleError');

    heroService.getHeroes().then(() => {
      expect(heroService.handleError).toHaveBeenCalled();
      done();
    })
  });

  it('should return the heroes array from the promise when getHeroes succeeds', (done) => {
    setup(MockSuccesGetHeroesHttp);
    spyOn(heroService, 'handleError');

    heroService.getHeroes().then((heroes) => {
      expect(heroService.handleError).not.toHaveBeenCalled();
      expect(heroes).toEqual(MockHeroesArray);
      done();
    })
  });

  it('should return the hero based on passed in id from the promise when it succeeds', (done) => {
    setup(MockSuccesGetHeroesHttp);

    heroService.getHero(MockHero.id).then((hero) => {
      expect(hero).toEqual(MockHero);
      done();
    })
  });
});

class MockFailedGetHeroesHttp extends Http {
  constructor(backend, options) {
    super(backend, options)
  }

  get() {
    return Observable.throw('error');
  }
}

class MockSuccesGetHeroesHttp extends Http {
  constructor(backend, options) {
    super(backend, options)
  }

  get() {
    return Observable.from([ new Response(new ResponseOptions({body: {data: MockHeroesArray}})) ]);
  }
}

let setup = function (httpMock) {
  TestBed.configureTestingModule({
    providers: [
      HeroService,
      MockBackend,
      BaseRequestOptions,
      {
        provide: Http,
        useFactory: (backend: MockBackend, options: BaseRequestOptions) => new httpMock(backend, options),
        deps: [ MockBackend, BaseRequestOptions ]
      }
    ]
  });
  inject([ MockBackend, Http ],
    (mb: MockBackend, http: Http) => {
      mockBackend = mb;
      heroService = new HeroService(http);
    })();
};
{% endhighlight %}

[e2e]: http://www.zackarychapple.guru/angular2/2016/10/19/angular2-e2e-testing.html
