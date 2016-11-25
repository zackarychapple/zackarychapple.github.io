---
layout: post
title:  "Angular2 Testing Components with Tour of Heroes"
date:   2016-11-25 11:40:56 -0500
categories: angular2
---
## Intro

Continuing on our journey of testing the Tour of Heroes app its time for us to take a look at testing components.

Here is the full [Tour of Heroes][e2e] app tested.

### heroes.component.ts

{% highlight javascript %}
@Component({
  selector: 'my-heroes',
  templateUrl: 'heroes.component.html',
  styleUrls: [ 'heroes.component.css' ]
})
export class HeroesComponent implements OnInit {
  heroes: Hero[];
  selectedHero: Hero;
  addingHero = false;
  error: any;

  constructor(private heroService: HeroService,
              private router: Router) {
  }

  getHeroes(): Promise<any> {
    return this.heroService
      .getHeroes()
      .then(heroes => this.heroes = heroes)
      .catch(error => this.error = error);
  }

  addHero(): void {
    this.addingHero = true;
    this.selectedHero = null;
  }

  close(savedHero: Hero): void {
    this.addingHero = false;
    if (savedHero) {
      this.getHeroes();
    }
  }

  deleteHero(hero: Hero, event: any): Promise<any> {
    event.stopPropagation();
    return this.heroService
      .delete(hero)
      .then(() => {
        this.heroes = this.heroes.filter((indexHero: Hero) => indexHero !== hero);
        if (this.selectedHero === hero) {
          this.selectedHero = null;
        }
      })
      .catch(error => this.error = error);
  }

  ngOnInit(): void {
    this.getHeroes();
  }

  onSelect(hero: Hero): void {
    this.selectedHero = hero;
    this.addingHero = false;
  }

  gotoDetail(): void {
    this.router.navigate([ '/detail', this.selectedHero.id ]);
  }
}

{% endhighlight %}

## Testing Setup

Similar to how we tested our services we need to setup our testing module for testing our components.
There are a couple of key differences though.  The first big difference is we are actually going to create an element
fixture.  To do this we need to create the component.  We can use the TestBed (which we already configured) to create
 the component.  The fixture is what we are going to use for our presentational testing. The component instance in the
 second beforeEach block is what we will use for our functional testing.
{% highlight javascript %}
beforeEach(() => {
  TestBed.configureTestingModule({
    providers: [
      HeroService,
      MockBackend,
      BaseRequestOptions,
      {
        provide: Http,
        useFactory: (backend: MockBackend, options: BaseRequestOptions) => new Http(backend, options),
        deps: [ MockBackend, BaseRequestOptions ]
      }
    ],
    declarations: [
      HeroesComponent,
      HeroDetailComponent
    ],
    imports: [
      FormsModule,
      RouterTestingModule
    ]
  });
  elementFixture = TestBed.createComponent(HeroesComponent);
});
beforeEach(inject([ HeroService, MockBackend, Router ],
  (hs: HeroService, mb: MockBackend, r: Router) => {
    heroService = hs;
    router = r;
    heroSearchComponent = new HeroesComponent(hs, r);
    mockBackend = mb;
  }));
{% endhighlight %}


## Testing Functionality

Once the setup is complete the functionality testing is actually very straight forward.

{% highlight javascript %}
it('should call getHeroes and set heroes to the returned object', (done) => {
  spyOn(heroService, 'getHeroes').and.callFake(() => {
    return Promise.resolve(MockHeroesArray);
  });

  heroSearchComponent.getHeroes().then(() => {
    expect(heroService.getHeroes).toHaveBeenCalled();
    expect(heroService.getHeroes).toHaveBeenCalledTimes(1);
    expect(heroSearchComponent.heroes).toBe(MockHeroesArray);
    done();
  });
});
{% endhighlight %}

## Testing Presentation

Testing presentation is something that can come in very handy.

When you are trying to verify the presentation and you make changes to the componentInstance make sure you call detect changes.

Note:  Make sure when you are making changes on the componentInstance you are making changes on the instance for the fixture.

{% highlight javascript %}
beforeEach(() => {
 elementFixture.componentInstance.heroes = MockHeroesArray;
 elementFixture.detectChanges();
});
{% endhighlight %}

After the component changes are detected we can grab the native element off of the fixture and verify our criteria.

{% highlight javascript %}
it('should have 2 hero-element\'s when heroes is populated', () => {
  heroesElement = elementFixture.nativeElement;
  expect(heroesElement.querySelectorAll('.hero-element').length).toBe(MockHeroesArray.length);
});
{% endhighlight %}

When testing functionality we can spyOn function calls and verify side effects.

{% highlight javascript %}
it('add the selected class to the selected hero and not other heroes', () => {
  heroesElement = elementFixture.nativeElement;
  spyOn(elementFixture.componentInstance, 'onSelect').and.callThrough();

  heroesElement.querySelectorAll('.hero-element')[ 0 ].click();

  elementFixture.detectChanges();
  let updatedElement = elementFixture.nativeElement;
  expect(elementFixture.componentInstance.onSelect).toHaveBeenCalled();
  expect(elementFixture.componentInstance.onSelect).toHaveBeenCalledTimes(1);
  expect(updatedElement.querySelectorAll('.hero-element')[ 0 ].parentNode.classList).toContain('selected');
  expect(updatedElement.querySelectorAll('.hero-element')[ 1 ].parentNode.classList).not.toContain('selected');
});
{% endhighlight %}

Here is the full tests.

{% highlight javascript %}
describe('Component: HeroSearch', () => {
  let elementFixture: ComponentFixture<HeroesComponent>;
  let heroService: HeroService;
  let mockBackend: MockBackend;
  let heroSearchComponent: HeroesComponent;
  let router: Router;
  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        HeroService,
        MockBackend,
        BaseRequestOptions,
        {
          provide: Http,
          useFactory: (backend: MockBackend, options: BaseRequestOptions) => new Http(backend, options),
          deps: [ MockBackend, BaseRequestOptions ]
        }
      ],
      declarations: [
        HeroesComponent,
        HeroDetailComponent
      ],
      imports: [
        FormsModule,
        RouterTestingModule
      ]
    });
    elementFixture = TestBed.createComponent(HeroesComponent);
  });
  beforeEach(inject([ HeroService, MockBackend, Router ],
    (hs: HeroService, mb: MockBackend, r: Router) => {
      heroService = hs;
      router = r;
      heroSearchComponent = new HeroesComponent(hs, r);
      mockBackend = mb;
    }));
  describe('Functional: ', () => {
    it('should call getHeroes and set heroes to the returned object', (done) => {
      spyOn(heroService, 'getHeroes').and.callFake(() => {
        return Promise.resolve(MockHeroesArray);
      });

      heroSearchComponent.getHeroes().then(() => {
        expect(heroService.getHeroes).toHaveBeenCalled();
        expect(heroService.getHeroes).toHaveBeenCalledTimes(1);
        expect(heroSearchComponent.heroes).toBe(MockHeroesArray);
        done();
      });
    });

    it('should call getHeroes and set heroes to the returned object', (done) => {
      const errorMsg = 'Some error';
      spyOn(heroService, 'getHeroes').and.callFake(() => {
        return Promise.reject(errorMsg);
      });
      heroSearchComponent.getHeroes().then(() => {
        expect(heroService.getHeroes).toHaveBeenCalled();
        expect(heroService.getHeroes).toHaveBeenCalledTimes(1);
        expect(heroSearchComponent.error).toBe(errorMsg);
        done();
      });
    });

    it('should call deleteHeroes and delete hero from hero\'s array and set selected to ' +
      'null when the hero passed was the selected hero', (done) => {
      const errorMsg = 'Some error';
      spyOn(heroService, 'delete').and.callFake(() => {
        return Promise.resolve(errorMsg);
      });
      heroSearchComponent.heroes = MockHeroesArray;
      heroSearchComponent.selectedHero = MockHero;
      heroSearchComponent.deleteHero(MockHero, MockEvent).then(() => {
        expect(heroService.delete).toHaveBeenCalled();
        expect(heroService.delete).toHaveBeenCalledTimes(1);
        expect(heroService.delete).toHaveBeenCalledWith(MockHero);
        expect(heroSearchComponent.heroes).toEqual([ MockHero2 ]);
        expect(heroSearchComponent.selectedHero).toBeNull();
        done();
      });
    });

    it('should call deleteHeroes and delete hero from hero\'s array and not set selected to ' +
      'null when the hero passed was different than selected hero', (done) => {
      const errorMsg = 'Some error';
      spyOn(heroService, 'delete').and.callFake(() => {
        return Promise.resolve(errorMsg);
      });
      heroSearchComponent.heroes = MockHeroesArray;
      heroSearchComponent.selectedHero = MockHero2;
      heroSearchComponent.deleteHero(MockHero, MockEvent).then(() => {
        expect(heroService.delete).toHaveBeenCalled();
        expect(heroService.delete).toHaveBeenCalledTimes(1);
        expect(heroService.delete).toHaveBeenCalledWith(MockHero);
        expect(heroSearchComponent.heroes).toEqual([ MockHero2 ]);
        expect(heroSearchComponent.selectedHero).toBe(MockHero2);
        done();
      });
    });

    it('should catch if an error is thrown at delete', (done) => {
      const errorMsg = 'some error';
      heroSearchComponent.heroes = MockHeroesArray;
      spyOn(heroService, 'delete').and.callFake(() => {
        return Promise.reject(errorMsg);
      });

      heroSearchComponent.deleteHero(MockHero, MockEvent).then(() => {
        expect(heroService.delete).toHaveBeenCalled();
        expect(heroService.delete).toHaveBeenCalledTimes(1);
        expect(heroService.delete).toHaveBeenCalledWith(MockHero);
        expect(heroSearchComponent.heroes).toEqual(MockHeroesArray);
        expect(heroSearchComponent.error).toBe(errorMsg);
        done();
      });
    });

    it('should switch to add hero mode and clear selected hero when addHero is called', () => {
      expect(heroSearchComponent.addingHero).toBeFalsy();
      heroSearchComponent.selectedHero = MockHero;

      heroSearchComponent.addHero();

      expect(heroSearchComponent.addingHero).toBeTruthy();
      expect(heroSearchComponent.selectedHero).toBeNull();
    });


    it('should expect not to ball getHeroes when savedHero is null', () => {
      spyOn(heroSearchComponent, 'getHeroes');
      heroSearchComponent.addingHero = true;
      expect(heroSearchComponent.addingHero).toBeTruthy();

      heroSearchComponent.close(null);
      expect(heroSearchComponent.addingHero).toBeFalsy();
      expect(heroSearchComponent.getHeroes).not.toHaveBeenCalled();
    });

    it('should switch from add hero mode', () => {
      spyOn(heroSearchComponent, 'getHeroes');
      heroSearchComponent.addingHero = true;
      expect(heroSearchComponent.addingHero).toBeTruthy();

      heroSearchComponent.close(MockHero);

      expect(heroSearchComponent.addingHero).toBeFalsy();
      expect(heroSearchComponent.getHeroes).toHaveBeenCalled();
      expect(heroSearchComponent.getHeroes).toHaveBeenCalledTimes(1);
    });

    it('should initialize and call getHeroes', () => {
      spyOn(heroSearchComponent, 'getHeroes');

      heroSearchComponent.ngOnInit();

      expect(heroSearchComponent.getHeroes).toHaveBeenCalled();
      expect(heroSearchComponent.getHeroes).toHaveBeenCalledTimes(1);
    });

    it('should set selected hero to the hero passed to onSelect', () => {
      heroSearchComponent.onSelect(MockHero);

      expect(heroSearchComponent.selectedHero).toBe(MockHero);
      expect(heroSearchComponent.addingHero).toBeFalsy();
    });

    it('should navigate to detail page for hero based on selected hero id', () => {
      spyOn(router, 'navigate');
      heroSearchComponent.selectedHero = MockHero;

      heroSearchComponent.gotoDetail();

      expect(router.navigate).toHaveBeenCalledWith([ '/detail', MockHero.id ]);
    });
  });

  describe('Presentation:', () => {
    let heroesElement;
    beforeEach(() => {
      elementFixture.componentInstance.heroes = MockHeroesArray;
      elementFixture.detectChanges();
    });
    it('should have 2 hero-element\'s when heroes is populated', () => {
      heroesElement = elementFixture.nativeElement;
      expect(heroesElement.querySelectorAll('.hero-element').length).toBe(MockHeroesArray.length);
    });
    it('add the selected class to the selected hero and not other heroes', () => {
      heroesElement = elementFixture.nativeElement;
      spyOn(elementFixture.componentInstance, 'onSelect').and.callThrough();

      heroesElement.querySelectorAll('.hero-element')[ 0 ].click();

      elementFixture.detectChanges();
      let updatedElement = elementFixture.nativeElement;
      expect(elementFixture.componentInstance.onSelect).toHaveBeenCalled();
      expect(elementFixture.componentInstance.onSelect).toHaveBeenCalledTimes(1);
      expect(updatedElement.querySelectorAll('.hero-element')[ 0 ].parentNode.classList).toContain('selected');
      expect(updatedElement.querySelectorAll('.hero-element')[ 1 ].parentNode.classList).not.toContain('selected');
    });

    it('deleted the selected hero and not other heroes', () => {
      heroesElement = elementFixture.nativeElement;
      spyOn(elementFixture.componentInstance, 'deleteHero').and.callFake((hero: Hero, $event: any) => {
        elementFixture.componentInstance.heroes.splice(elementFixture.componentInstance.heroes.indexOf(hero), 1);
      });

      heroesElement.querySelectorAll('.hero-element')[ 0 ].parentElement.querySelectorAll('.delete-button')[ 0 ].click();

      expect(elementFixture.componentInstance.heroes.length).toBe(1);
      elementFixture.detectChanges();
      let updatedElement = elementFixture.nativeElement;
      expect(elementFixture.componentInstance.deleteHero).toHaveBeenCalled();
      expect(elementFixture.componentInstance.deleteHero).toHaveBeenCalledTimes(1);
      expect(elementFixture.componentInstance.deleteHero).toHaveBeenCalledWith(MockHero, jasmine.anything());
      expect(updatedElement.querySelectorAll('.hero-element').length).toBe(1);
    });

    it('should display the error message when an error is set', () => {
      heroesElement = elementFixture.nativeElement;
      expect(heroesElement.querySelectorAll('.error').length).toBe(0);

      elementFixture.componentInstance.error = 'something happened';

      elementFixture.detectChanges();
      let updatedElement = elementFixture.nativeElement;
      expect(updatedElement.querySelectorAll('.error').length).toBe(1);
    });

  });
});
{% endhighlight %}

[e2e]: http://www.zackarychapple.guru/angular2/2016/10/19/angular2-e2e-testing.html
