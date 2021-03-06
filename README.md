<div align="center">
  <h1>React Prerendered Component</h1>
  <br/>
  Partial Hydration and Component-Level Caching 
  <br/>
  <br/>
  
  <a href="https://www.npmjs.com/package/react-prerendered-component">
    <img src="https://img.shields.io/npm/v/react-prerendered-component.svg?style=flat-square" />
  </a>
</div>

## Idea
In short: dont try to __run__ js code, and produce a react tree matching pre-rendered one,
but __use__ pre-rendered html until js code will be ready to replace it. Make it live.

What else could be done on HTML level? Caching, _templatization_, and other good things to 🚀, just in a 3kb*.

#### Prerendered component
>Render something on server, and use it as HTML on the client

- Server side render data 
  - call `thisIsServer` somewhere, to setup enviroment.
  - React-prerendered-component `will leave trails`, wrapping each block with div with _known_ id.
- Hydrate the client side 
  - React-prerendered-component will search for _known_ ids, and `read rendered HTML` back from a page.
- You site is ready!
  - React-prerendered-components are ready. They are rendering a pre-existing HTML you send from a server.
- Once any component ready to be replaced - hydrate 
  - But not before. That's the point - partial hydration, step by step
  
Bonus - you can store and restore component state.

More details - https://twitter.com/theKashey/status/1021739948536295424

## Usage

1. __Restore__ data from HTML
```js
<PrerenderedComponent
  // restore - access DIV and get "counter" from HTML
  restore={(el) => this.setState({counter: +el.querySelector('i').innerHTML})}
  // once we read anything - go live!
  live={!!this.state.counter}
>
  <p>Am I alive?</p>
  <i>{this.props.counter}</i>
</PrerenderedComponent>
```

Is components HTML was not generated during SSR, and it would be not present in the page code - 
component will go `live` automatically, unless `strict` prop is set.

2. To do a __partial hydrating__

You may keep some pieces of your code as "raw HTML" until needed. This needed includes:
- code splitting. You may decrease time to __First Meaningful Paint_ code splitting JS code needed by some blocks, keeping those blocks visible.
Later, after js-chunk loaded, it will rehydrate existing HTML.
- deferring content below the fold. The same code splitting, where loading if component might be triggered using interception observer.

If your code splitting library (not React.lazy) supports "preload" - you may use it to control code splitting
```js
const AsyncLoadedComponent = loadable(() => import('./deferredComponent'));
const AsyncLoadedComponent = imported(() => import('./deferredComponent'));

<PrerenderedComponent
  live={AsyncLoadedComponent.preload()} // once Promise resolved - component will go live  
>
  <AsyncLoadedComponent />
</PrerenderedComponent>
```

3. Restore state from JSON stored among.
```js
<PrerenderedComponent
  // restore - access DIV and get "counter" from HTML
  restore={(_,state) => this.setState(state)}
  store={ this.state }
  // once we read anything - go live!
  live={!!this.state.counter}  
>
  <p>Am I alive?</p>
  <i>{this.props.counter}</i>
</PrerenderedComponent>
```

### This is not SSR friendly unless....
Wrap your application with `PrerenderedControler`. This would provide context for all nested components, and "scope" _counters_ used to
represent nodes.

This is more Server Side requirement.
```js
import {PrerenderedControler} from 'react-prerendered-component';

ReactDOM.renderToString(<PrerenderedControler><App /></PrerenderedControler>);
```

> !!!!

Without `PrerenderedControler` SSR will always produce an unique HTML, you will be not able to match on Client Side.

> !!!!

## Client side-only components
It could be a case - some components should live only client-side, and completely skipped during SSR.
```js
import {render} from 'react-dom';
import { ClientSideComponent } from "react-prerendered-component";

const controller = cacheControler(cache);

const result = render(
  <ClientSideComponent>
     Will be rendered only on client
  </ClientSideComponent>
);

const result = hydrate(
  <PrerenderedControler hydrated>
      <ClientSideComponent>
         Will be rendered only on client __AFTER__ hydrate
      </ClientSideComponent>
  </PrerenderedControler>
);  
```

- There is the same component for `ServerSideComponent`s
- There are _hoc_ version for both cases
```js
import {clientSideComponent} from 'react-prerendered-component';

export default clientSideComponent(MyComponent);
```

## Safe SSR-friendly code splitting
Partial rehydration could benefit not only SSR-enhanced applications, but provide a better experience for simple code splitting.

In the case of SSR it's quite important, and quite hard, to load all the used chunks before
triggering `hydrate` method, or some _unloaded_ parts would be replaced by "Loaders".

Preloaded could help here, if your code-splitting library support `preloading`, __even__ if it does not support SSR.
```js
import imported from 'react-imported-component';
import {PrerenderedComponent} from "react-prerendered-component";

const AsyncComponent = imported(() => import('./myComponent.js'));

<PrerenderedComponent
  // component will "go live" when chunk loading would be done
  live={AsyncComponent.preload()}
>
  // until component is not "live" prerendered HTML code would be used
  // that's why you need to `preload`
  <AsyncComponent/>
</PrerenderedComponent>
```
Yet again - it works with any library which could `preload`, which is literally any library except
`React.lazy`.

## Caching
Prerendered component could also work as a component-level cache.
Component caching is completely safe, compatible with any React version, but - absolutely
synchronous, thus no Memcache or Redis are possible.
 
```js
import {renderToString, renderToNodeStream} from 'react-dom/server';
import {
  PrerenderedControler, 
  cacheControler, 
  CachedLocation, 
  cacheRenderedToString, 
  createCacheStream
} from "react-prerendered-component";

const controller = cacheControler(cache);

const result = renderToString(
  <PrerenderedControler control={control}>
     <CachedLocation cacheKey="the-key">
        any content
     </CachedLocation>
  </PrerenderedControler>
)

// DO NOT USE result! It contains some system information  
result === <x-cachedRANDOM_SEED-store-1>any content</x-cachedRANDOM_SEED-store-1>

// actual caching performed in another place
const theRealResult = cacheRenderedToString(result);

theRealResult  === "any content";


// Better use streams
renderToNodeStream(
  <PrerenderedControler control={control}>
     <CachedLocation cacheKey="the-key">
        any content
     </CachedLocation>
  </PrerenderedControler>
)
.pipe(createCacheStream(control)) // magic here
.pipe(res)
```
Stream API is completely _stream_ and would not delay Time-To-First-Byte

- `PrerenderedControler` - top level controller for a cache. Requires `controler` to be set
- `CachedLocation` - location to be cached. 
  - `cacheKey` - string - they key
  - `ttl` - number - time to live
  - `refresh` - boolean - flag to ignore cache
  - `clientCache` - boolean - flag to enable cache on clientSide (disabled by default)
  - `noChange` - boolean - disables cache at all
  - `variables` - object - varibles to use in templatization
  
  - `as=span` - string, used only for client-side cache to define a `wrapper` tag
  - `className` - string, used only for client-side cache
  - `rehydrate` - boolean, used only for client-side cache, false values would keep content as a _dead_ html.
- `Placeholder` - a template value
  - `name` - a variable name
- `WithPlaceholder` - renderprop version of `Placeholder`   
- `NotCacheable` - mark location as non-cacheable, preventing memoization    
- `cacheControler(cache)` - a cache controller factor, requires object with `cache` interface to work.
  - cache interface is `{ get(key): string, set(key, ttl):void }`
  - cache implimentation is NOT provided by this library.

#### Placeholder and Templatization
To optimize rendering performance and reduce memory usage you might use cache _templates_:
```js
import {CachedLocation, Placeholder, WidthPlaceholder} from 'react-prerendered-component';

const Component = () => (
  <CachedLocation key="myKey" variabes={{name: "GitHub", secret: 42 }}>
    the <Placeholder name="name"/>`s secret is <Placeholder name="secret"/>
    // it's easy to use placeholders in a plain HTML 
    
    <WithPlaceholder>
    { placeholder => (
       <img src="./img.jpg" alt={placeholder("name") + placeholder("secret")}/>
       // but to use it in "string" attribures you have to use render props
    )}
    </WithPlaceholder>
  </CachedLocation>
)
```   
    
#### NotCacheable
Sometimes you might got something, which is not cacheable. 
Sometimes cos you better not cache like - like personal information.
Sometimes cos it reads data from variable sources and could not be "just cached".
It is always __hard to manage__ it. So - just dont cache. It's a one line fix.

```js
import {NotCacheable, notCacheable} from 'react-prerendered-component';

const SafeCache = () => (
  <NotCacheable>
    <YourComponent />
  </NotCacheable>
);

const SafeComponent = notCacheable(YourComponent);
```
  
#### Sharing cache between multiple process
Any network based caches are not supported, the best cache you can use - LRU, is bound to single
process, while you probably want multi-threaded(workers) rendering, but dont want to maintain 
per-instance cache.

You may use nodejs shared-memory libraries (not supported by nodejs itself), like:
 - https://github.com/allenluce/mmap-object 

#### Cache speed
Results from rendering a single page 1000 times. All tests executed twice to mitigate possible v8 optimizations.
```text
dry      1013 - dry render to kick off HOT
base     868  - the __real__ rendering speed, about 1.1ms per page
cache    805  - with `cacheRenderedToString` used on uncachable appp
cache    801  - second run (probably this is the "real" speed)
partial  889  - with `cacheRenderedToString` used lightly cached app (the cost of caching)
partial  876  - second run
half     169  - page content cached
half     153  - second run
full     22   - full page caching
full     19   - second run
```
- full page cache is 42x faster. 0.02ms per page render
- half page render is 5x faster.
- partial page render is 1.1x slower.

#### Prerendered support
It is __safe__ to have `prerendered` component inside a cached location.


### Additional API
1. `ServerSideComponent` - component to be rendered only on server. Basically this is PrerenderedComponent with `live=false`
2. `ClientSideComponent` - component to be rendered only on client. Some things are not subject for SSR.
`ClientSideComponent` would not be initially rendered with `hydrated` prop enabled 
3. `thisIsServer(flag)` - override server/client flag
4. `isThisServer()` - get current environment.

## Automatically goes live
Prerendered component is work only once. Once it mounted for a first time. 

Next time another UID will be generated, and it will not find component to match.
If prerendered-component could not find corresponding component - it goes live automatically.

## Testing
Idea about PrerenderedComponent is to render something, and rehydrate it back. You should be able to 
render the same, using rehydrated data.
- render
- restore
- render
- compare. If result is equal - you did it right.

## While area is not "live" - it's dead
Until component go live - it's dead HTML code. You may be make it more alive by
transforming HTML to React, using [html-to-react](https://github.com/aknuds1/html-to-react),
and go live in a few steps.

## Size 🤯
Is this package 25kb? What are you thinking about?

- no, __this package is just 3kb__ or less - tree shaking is great (but not in a dev mode)
It __is__ bigger only on server.  

## See also
[react-progressive-hydration](https://github.com/GoogleChromeLabs/progressive-rendering-frameworks-samples/tree/master/react-progressive-hydration) - Google IO19 demo is quite similar to `react-prerendered-component`, but has a few differences.
- does not use _stable uids_, utilizing `__html:''` + `sCU` hack to prevent HTML update
- uses `React.hydrate` to make component live, breaking connectivity between React Trees
- while it has some benefits (no real HTML update), it might not be production ready right now 

## Licence
MIT
