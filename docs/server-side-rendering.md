# Five steps to enable Server-Side Rendering in your [Elmish](https://github.com/fable-elmish/elmish) + [DotNet Core](https://github.com/dotnet/core) App!

> [SSR Sample App](https://github.com/fable-compiler/fable-react/tree/master/Samples/SSRSample) based on [SAFE-Stack](https://github.com/SAFE-Stack/SAFE-BookStore) template is available!

## Introduction

### What is Server-Side Rendering (SSR) ?

Commonly speaking SSR means the majority of your app's code can run on both the server and the client, it is also as known as "isomorphic app" or "universal app". In React, you can render your components to html on the server side (usually a nodejs server) by `ReactDOMServer.renderToString`, reuse the server-rendered html and bind events on the client side by `React.hydrate`.

#### Pros

* Better SEO, as the search engine crawlers will directly see the fully rendered page.
* Faster time-to-content, especially on slow internet or slow devices.

#### Cons

* Development constraints, browser-specific code need add compile directives to ignore in the server.
* More involved build setup and deployment requirements.
* More server-side load.

#### Conclusions

While SSR looks pretty cool, it still adds more complexity to your app, and increases server-side load. But it could be really helpful in some cases like solving SEO issue in SPAs, improving time-to-content of mobile sites, etc.

### Server-Side Rendering in fable-react

fable-react's SSR approach is a little different from those you see on the network, it is a **Pure F#** approach. It means you can render your elmish's view function directly on dotnet core, with all benefits of dotnet core runtime!

There are lots of articles about comparing dotnet core and nodejs, I will only mention two main differences between F#/dotnet core and nodejs in SSR:

* F# is a compiled language, which means it's generally considered faster then a dynamic language, like js.
* Nodejs's single thread, event-driven, non-blocking I/O model works well in most web sites, but it is not good at CPU intensive tasks, including html rendering. Usually we need to run multi nodejs instances to take the advantage of multi-core systems. DotNet support non-blocking I/O (and `async/await` sugar), too. But the awesome part is that it also has pretty good support for multi-thread programming.

In a simple test, rendering on dotnet core is about ~1.5x faster then nodejs (with ReactDOMServer.renderToString + NODE_ENV=production) in a single thread. You can find more detail in the bottom of this page.

In a word, with this approach, you can not only get a better performance then nodejs, but also don't need the complexity of running and maintaining nodejs instances on your server!

Here is a list of Fable.Helpers.React API that support server-side rendering:

* HTML/CSS/SVG DSL function/unions, like `div`, `input`, `Style`, `Display`, `svg`, etc.
* str/ofString/ofInt/ofFloat
* ofOption/ofArray/ofList
* fragment
* ofType
* ofFunction

These don't support, but you can wrap it by `Fable.React.Isomorphic.isomorphicView` to skip or render a placeholder on the server:

* ofImport

## Step 1: Reorganize your source files

Separate all your elmish view and types to standalone files, like this:

```F#

pages
|-- Home
    |-- View.fs // contains view function.
    |-- Types.fs // contains msg and model type definitions, also should include init function.
    |-- State.fs // contains update function

```

View.fs and Types.fs will be shared between client and server.

## Step 2. Make sure shared files can be executed on the server side

Some code that works in Fable might throw a runtime exception on dotnet core, we should be careful with unsafe type casting and add compiler directives to remove some code if necessary.

Here are some hints about doing this:

### 1. Replace unsafe cast (unbox and `!!`) in your HTML attributes and CSS props with `HTMLAttr.Custom`, `SVGAttr.Custom` and `CSSProp.Custom`

```diff
- div [ !!("class", "container") ] []
+ div [ HTMLAttr.Custom ("class", "container") ]


- div [ Style [ !!("class", "container") ] ] []
+ div [ Style [ CSSProp.Custom("class", "container") ] ] []

- svg [ !!("width", 100) ] []
+ svg [ SVGAttr.Custom("class", "container") ] []
```


### 2. Make sure your browser/js code won't be executed on the server side

One big challenge of sharing code between client and server is that the server side has different API environment with client side. In this respect Fable + dotnet core's SSR is not much different than nodejs, except on dotnet core you should not only prevent browser's API call, but also js.

Thanks for Fable Compiler's `FABLE_COMPILER` directive, we can easily distinguish it's running on client or server and execute different code in different environment:

```#F
#if FABLE_COMPILER
    executeOnClient ()
#else
    executeOnServer ()
#endif
```

We also provide a help function in `Fable.Helpers.Isomorphic`, the definition is:

```F#
let inline isomorphicExec clientFn serverFn input =
#if FABLE_COMPILER
    clientFn input
#else
    serverFn input
#endif
```

Full example:

```diff
open Fable.Core
open Fable.Core.JS
open Fable.React.Isomorphic
open Browser

// example code to add marquee effect to your document's title
-window.setInterval(
-    fun () ->
-        document.title <- document.title.[1..len - 1] + document.title.[0..0],
-    600
-)


+let inline clientFn () =
+    window.setInterval(
+        fun () ->
+            document.title <- document.title.[1..len - 1] + document.title.[0..0],
+        600
+    )
+isomorphicExec clientFn ignore ()
```


### 3. Add a placeholder for components that cannot been rendered on the server side, like js native components.

In `Fable.React.Isomorphic` we also implemented a help function (`isomorphicView`) to render a placeholder element for components that cannot be rendered on the server side, this function will also help [React.hydrate](https://reactjs.org/docs/react-dom.html#hydrate) to understand the differences between htmls rendered by client and server, so React won't treat it as a mistake and warn about it.

```diff
open Fable.Core
open Fable.Core.JS
open Fable.React
open Fable.React.Isomorphic
open Browser

type JsCompProps = {
  text: string
}

let jsComp (props: JsCompProps) =
  ofImport "default" "./jsComp" props []

-jsComp { text="I'm rendered by a js Component!" }

+let jsCompServer (props: JsCompProps) =
+  div [] [ str "loading" ]
+
+isomorphicView jsComp jsCompServer { text="I'm rendered by a js Component!" }
```

## Step 3. Create your initial state on the server side.

On the server side, you can create routes like normal MVC app, just make sure the model passed to server-side rendering function is exactly match the model on the client side in current route.

Here is an example:

```F#

open Giraffe
open Giraffe.GiraffeViewEngine
open FableJson

let initState: Model = {
    counter = Some 42
    someString = "Some String"
    someFloat = 11.11
    someInt = 22
}

let renderHtml () =
    // This would render the html by model create on the server side.
    // Note in an Elmish app, view function takes two parameters,
    // the first is model, and the second is dispatch,
    // which simple ignored here because React will bind event handlers for you on the client side.
    let htmlStr = Fable.Helpers.ReactServer.renderToString(Client.View.view initState ignore)

    // We also need to pass the model to Elmish and React by print a json string in html to let them know what's the model that used to rendering the html.
    // Note we call ofJson twice here,
    // because Elmish's model can contains some complicate type instead of pojo,
    // the first one will seriallize the state to json string,
    // and the second one will seriallize the json string to a legally js string,
    // so we can deseriallize it by Fable's ofJson and get the correct types.
    let stateJsonStr = toJson (toJson initState)

    html []
        [ head [] []
        body []
            [ div [_id "elmish-app"] [ rawText htmlStr ]
            script []
                [ rawText (sprintf """
                var __INIT_STATE__ = %s
                """ stateJsonStr) ] //
            script [ _src (assetsBaseUrl + "/public/bundle.js") ] []
            ]
        ]
```

## Step 4. Update your elmish app's init function

1. Initialize your elmish app by state printed in the HTML.
2. Remove initial commands that fetch state which already included in the HTML.

e.g.

```F#
let init () =
  // Init model by server side state
  let model = ofJson<Model> !!window?__INIT_STATE__
  // let cmd =
  //   Cmd.ofPromise
  //     (fetchAs<int> "/api/init")
  //     []
  //     (Ok >> Init)
  //     (Error >> Init)
  model, Cmd.none
```

## Step 5. Using React.hydrate to render your app

```diff
Program.mkProgram init update view
#if DEBUG
|> Program.withConsoleTrace
|> Program.withHMR
#endif
-|> Program.withReact "elmish-app"
+|> Program.withReactHydrate "elmish-app"
#if DEBUG
|> Program.withDebugger
#endif
|> Program.run
```

Now enjoy! If you find bugs or just need some help, please create an issue and let us know, thanks!

## Try the sample app

```sh
git clone https://github.com/fable-compiler/fable-react.git
cd ./fable-react/Samples/SSRSample/
./build.sh run # or ./build.cmd run on windows
```

## Run simple benchmark test in sample app

The SSRSample project also contains a simple benchmark test, you can try it in you computer by:

```sh

cd ./Samples/SSRSample
./build.sh bench # or ./build.cmd bench on windows

```

Updated benchmark (MacBookPro 2019 w/ Core i9 2.4 GHz), .net5.0 & node.js 14.4.0
```sh

dotnet ./bin/Release/net5.0/dotnet.dll
Thread 1 started
Thread 1 render 320000 times used 70954ms
[Single thread] 70954ms    4509.964req/s
Thread 1 started
Thread 4 started
Thread 5 started
Thread 6 started
Thread 7 started
Thread 8 started
Thread 9 started
Thread 10 started
Thread 12 started
Thread 11 started
Thread 13 started
Thread 15 started
Thread 14 started
Thread Thread 17 started
18 startedThread 
19 started
Thread 14 render 20000 times used 15288ms
Thread 19 render 20000 times used 15372ms
Thread 1 render 20000 times used 15478ms
Thread 9 render 20000 times used 15485ms
Thread 10 render 20000 times used 15527ms
Thread 8 render 20000 times used 15566ms
Thread 5 render 20000 times used 15578ms
Thread 7 render 20000 times used 15581ms
Thread 6 render 20000 times used 15594ms
Thread 4 render 20000 times used 15605ms
Thread 18 render 20000 times used 15611ms
Thread 11 render 20000 times used 15618ms
Thread 13 render 20000 times used 15623ms
Thread 17 render 20000 times used 15646ms
Thread 15 render 20000 times used 15656ms
Thread 12 render 20000 times used 15685ms
[16 tasks] Total: 15557ms    Memory footprint: 46.672MB   Requests/sec: 20569.519



/usr/local/bin/node ./node.js
Master 32903 is running
[Single process] 23541ms   render 320000 times  13593.305req/s
Worker 32908: started
Worker 32909: started
Worker 32912: started
Worker 32911: started
Worker 32913: started
Worker 32914: started
Worker 32910: started
Worker 32915: started
Worker 32916: started
Worker 32918: started
Worker 32917: started
Worker 32920: started
Worker 32919: started
Worker 32921: started
Worker 32922: started
Worker 32923: started
Worker 32911: render 20000 times used 4127ms
Worker 32916: render 20000 times used 4147ms
Worker 32923: render 20000 times used 4125ms
Worker 32917: render 20000 times used 4177ms
Worker 32913: render 20000 times used 4194ms
Worker 32909: render 20000 times used 4209ms
Worker 32910: render 20000 times used 4196ms
Worker 32920: render 20000 times used 4178ms
Worker 32912: render 20000 times used 4211ms
Worker 32915: render 20000 times used 4196ms
Worker 32921: render 20000 times used 4164ms
Worker 32914: render 20000 times used 4206ms
Worker 32922: render 20000 times used 4169ms
Worker 32919: render 20000 times used 4191ms
Worker 32908: render 20000 times used 4232ms
Worker 32918: render 20000 times used 4230ms
[16 workers] Total: 4184.5ms    Memory footprint: 387.550MB    Requests/sec: 76472.697


```
