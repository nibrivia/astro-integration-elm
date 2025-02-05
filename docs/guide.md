# Elm Integration Guide

## API Guide & Compiler Configuration

The API for the `astro-integration-elm` package is very minimal, as only one function, the integration, is exported. The typescript typings for this are pretty weak currently, but the arguments are just forwarded to [rtfeldman/node-elm-compiler](https://github.com/rtfeldman/node-elm-compiler).

The integration will automatically run the Elm compiler in optimised mode when you run a full (non dev) build. You can override this, or add other Elm compiler settings to the integration as parameters to the `elm()` function.

## Using Flags

Astro `props` map very neatly onto Elm `flags`, and an Astro component (or island) is a good fit for an Elm `element`. The integration simply calls the `init` function, which can take the props as either:

- A `Json.Decode.Value`, which you can **safely** de-serialize as you would regular JSON.
- A custom record type, which will be **unsafely** converted from JSON to an Elm type (automatically).

Both of these approaches look the same from the Astro side, which one you are using depends on the type of your `init` function:

```elm
import Json.Decode

init : Decode.Value -> ( Model, Cmd msg )

-- vs --

init : { class : Maybe String, count : Int } -> ( Model, Cmd msg )
```

Using a custom record type is **dangerous** if you're server side rendering: if the wrong types are passed, your page will not render for that request. However, if you're generating a static site, it's safe to use record types - an issue with the flag types will only cause a "build time error".

## Default Flags (`class` & `server`)

The integration automatically adds a flag, called `"server"`, which is set to `true` when your component is being rendered on the server and false if on the client-side.

If a `<style>` tag is included in the parent Astro component, astro will pass a `class` prop to the Elm component - you can access this like any other flag.

```html
<style>
  #my-elm-component {
    font-style: italic;
    border: 1px solid red;
  }
</style>
<StyledComponent client:load />
```

```elm
module StyledComponent exposing (..)

import Html exposing (div)
import Html.Attributes exposing (class, id)

init : { class : String } -> ( { class : String }, Cmd msg )

-- ...

view model =
  div
    [ class model.class
    , id "my-elm-component"
    ]
    [ -- ...
    ]
```

## More Complex Example: Props & `client:load`

_index.astro_

```jsx
---
import Counter from "../components/Counter.elm";
---
<html>
  <body>
    <main>
      <h1>Astro and Elm:</h1>
      <Counter client:load count={99} />
    </main>
  </body>
</html>

<style>
  :root {
    font-family: Helvetica, Arial, sans-serif;
  }

  main {
    margin: auto;
    padding: 1em;
    max-width: 60ch;
  }
</style>
```

_Counter.elm_

```elm
module Counter exposing (main)

import Browser
import Html exposing (button, div, text)
import Html.Events exposing (onClick)
import Json.Decode as D

type alias Model =
    Int

main =
    Browser.element
        { init = init
        , view = view
        , update = update
        , subscriptions = \_ -> Sub.none
        }


init : D.Value -> ( Model, Cmd msg )
init flags =
    ( Result.withDefault 0 <| D.decodeValue flagsDecoder flags
    , Cmd.none
    )

flagsDecoder : D.Decoder Int
flagsDecoder =
        (D.field "count" D.int)

type Msg = Inc | Dec

update msg count =
    case msg of
        Inc -> (count + 1, Cmd.none)
        Dec -> (count - 1, Cmd.none)

view count =
    div
      [ text <| String.fromInt count
      , button [ onClick Inc ] [ text "+" ]
      , button [ onClick Dec ] [ text "-" ]
      ]
```

## Which kinds of Elm program can I include?

Elm's `Browser` module contains 4 functions to create an Elm program.

- `Browser.sandbox`
- `Browser.element`
- `Browser.document`
- `Browser.application`

You can `sandbox` & `element` with ease. Using `document` is more nuanced, as it takes over the DOM's `document.body` element. It will work, but it will remove all body content in your Astro file, so be careful! You can only use one `document` at a time, which I suppose makes sense.

The `application` function is not supported, as it is designed for SPAs, and doesn't appear to be a good fit for Astro. This may change in a future release.

## Advanced: Using `ports`

Elm's ports are a great way to use web APIs that aren't available in Elm.

However, Astro [doesn't allow passing functions to hybrid props](https://guide.elm-lang.org/interop/ports.html).

The (less than ideal) solution I offer is to use the `unsafeSetup` prop.

**This is highly dangerous!**

```jsx
<Test
  client:load
  // CALLED UNSAFE FOR A REASON: Please read the docs before use.
  unsafeSetup={`(app) => {
    var socket = new WebSocket('wss://echo.websocket.org')

    app.ports.sendMessage.subscribe(function(message) {
        socket.send(message);
    })

    socket.addEventListener("message", (event) => {
        app.ports.messageReceiver.send(event.data);
    })
  }`}
/>
```

Here we can establish an (incoming and outgoing) websocket connection to the server with the `unsafeSetup` function.

The `unsafeSetup` function is evaluated **in the client's browser** with `eval`.

This can be a **huge security vulnerability**. Do not allow interpolate user-generated, external or unescaped content into the `unsafeSetup` prop.

🚨 I repeat - **do not** put anything except trusted code in this string.

Much like Astro's [`set:html`](https://docs.astro.build/en/reference/directives-reference/#sethtml) or React's `dangerouslySetInnerHTML`, this opens you up to XSS attacks.
