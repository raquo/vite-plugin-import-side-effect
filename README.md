# Scala.js Import Side Effect Vite Plugin

This [Vite](https://vitejs.dev/) plugin lets you import resources for their side effects only, without capturing their value. Basically, it lets Scala.js make plain imports like `import "foo.css"` instead of `import * as unused from "foo.css"`, which is what JSImport does.

This is especially relevant to importing CSS resources, because if we do `JSImport("foo.css")`, which results in `import * as unused from "foo.css"`, Vite production build will include the resulting CSS into the JS bundle, but it will also generate a CSS file containing this CSS. So, CSS will be double-loaded, which is wasteful. It's also annoying to see duplicate CSS declarations in browser dev tools.

On the other hand, if we said `import "foo.css"`, Vite production build would only include the CSS in the CSS file, it would not add it to the JS bundle.

At the moment, Scala.js can't emit `import "foo.css"`, so this plugin does some unsightly things to simulate that. It's an improvement from _unspeakable things_ of v0.1 though.


## Usage

**This documentation is for v0.2.0 of this plugin**. v0.1.0 used a different approach, you can read about it [here](https://github.com/raquo/vite-plugin-import-side-effect/tree/900d11710735f4457dae7608372f3b9dda7b95fd) (and then upgrade).

```
npm i -D @raquo/vite-plugin-import-side-effect@0.2.0
```

**Without** this plugin, you would normally import CSS somehow like this in Scala.js:

```scala
/** Marks an import as "used" in Scala.js to prevent dead code elimination */
def useImport(obj: js.Object): Unit = ()

@JSImport("highlight.js/styles/dark.min.css", JSImport.Namespace)
@js.native private object Stylesheet extends js.Object

useImport(Stylesheet)
```

This works, but causes the double-loading of CSS mentioned above, as well as warnings from Vite which will become errors eventually.

This plugin supports this pattern, just add it to your Vite config:

```js
import importSideEffectPlugin from "@raquo/vite-plugin-import-side-effect";

// Add this to the list of Vite plugins:
importSideEffectPlugin({
  defNames: [], // see "Compact syntax" below
  rewriteModuleIds: ['**/*.css', '**/*.less'],
  verbose: true
})
```

This will rewrite all imports that match those glob patterns from `import * as unused from "moduleId"` to simply `import "moduleId"`.

The `verbose` option will print out the affected `moduleId`-s.

Technically, this transformation is applied to all .js files that aren't in `node_modules`, but the way our import matching is implemented, in practice it only works on JS files outputted by Scala.js. Which is as intended, non-Scala.js files don't need this transformation, as you can trivially write `import "foo.css"` in them.


### Compact syntax

I am so very annoyed by call-site boilerplate. I don't want to repeat `JSImport.Namespace`, I don't want to import or re-define `useImport` every time I want to import CSS. So, I use this plugin in the following way:

```scala
@JSImport("highlight.js/styles/dark.min.css")
@js.native private def importStyle(): Unit = js.native

importStyle() // Marks an import as "used" in Scala.js to prevent dead code elimination
```

To do this, you need to update the plugin's options in vite.config.js:

```js
importSideEffectPlugin({
  defNames: ["importStyle"], // Must match the `def` name you defined above !!!
  rewriteModuleIds: ['**/*.css', '**/*.less'],
  verbose: true
})
```

Note: If you specify `defNames` like this, you can keep using the standard Scala.js importing syntax, this just unlocks alternative compact syntax.

Here, we are abusing Scala.js syntax intended for a different purpose to shave off some boilerplate. Up to you if you want to do this, it's completely optional.

How it works: In Scala.js, this `JSImport` syntax that annotates a `def importStyle` is intended to import the `importStyle` function from JS module, and will output something like this:

```js
import * as foo from "highlight.js/styles/dark.min.css"
foo.importStyle() // Name of the method
```

`importStyle` is just a name I made up, it does not actually exist anywhere. But since our code is calling it (to mark the import as used, and avoid the need for `JsImport.Namespace`), we can't just remove `foo` from the generated JS code. So this plugin rewrites this import to something like this instead:

```js
const foo = {importStyle: function () {}}
import "highlight.js/styles/dark.min.css"
```

And this way, the `foo.importStyle()` call generated by Scala.js does not fail at runtime (and doesn't do anything, which is fine, all we wanted was to import CSS).

For this to be safe, you should choose a name that does not actually exist on the files that you import. Or to be more precise, it's even ok if it exists, as long as nothing in your code actually wants to import it, because you know, it won't be able to. CSS is imported as a string by default, so pretty much any name is safe.

If you need to make multiple imports in the same Scala file, you can change the def name to avoid the name conflict, while providing "importStyle` as an alias (acting like a @JSName, essentially) in the second argument to `JSImport`. This way you don't need to add more `defName`-s to your vite config:

```scala
@JSImport("highlight.js/styles/light.min.css", "importStyle")
@js.native private def importLightStyle(): Unit = js.native

importLightStyle()

@JSImport("highlight.js/styles/dark.min.css", "importStyle")
@js.native private def importDarkStyle(): Unit = js.native

importDarkStyle()
```

vite.config.js:

```js
importSideEffectPlugin({
  defNames: ["importStyle"], // Still the same
  rewriteModuleIds: ['**/*.css', '**/*.less'],
  verbose: true
})
```

### Other ways to achieve more compact syntax

If you don't want to abuse Scala.js syntax but still want more compact syntax, you have a shared trait that your components extend, you can do this:

```scala
trait Component {

  // Marks an import as "used" in Scala.js to prevent dead code elimination
  protected def useImport(obj: js.Any): Unit = ()
  
  protected def Stylesheet: js.Any
  
  useImport(Stylesheet)
}
```

and then do this in your components:

```scala
object MyComponent extends Component {

  @JSImport("/path/to/MyComponent.css", JSImport.Namespace)
  @js.native private object Stylesheet extends js.Object
}
```

If you do this, then you can keep the `defNames` array empty in the plugin's vite config.


## For Scala.js library authors

If you are writing a Scala.js library to be used by other applications, and it needs to import CSS, you shouldn't (and can't, really) depend on this plugin, because this plugin runs at the bundle building time, and you're not distributing a JS bundle, you're distributing SJSIR files. It's the end-users who are responsible for bundling their applications. Some aren't even using Vite.

I guess you have two options to include CSS:
- Tell the end user to import the necessary CSS themselves
- Use the normal Scala.js import (without the `importStyle` hack). I'm actually not sure if this would work.

In both cases, it's up to the users to make sure that their bundler can import your CSS. If their bundler is Vite, they'll need this plugin.


## Alternatives

If you want to use Vite, want to import CSS, but don't want to use this plugin, you can create a small JS file, write `import "foo.css"` in there (one or many), and then import that JS file to Scala.js, or import it into your `index.js` entry point using normal means.

You can also [@import](https://developer.mozilla.org/en-US/docs/Web/CSS/@import) one CSS file from another, instead of importing all CSS files into a JS file.

I used to do this, but it gets annoying when you have one CSS / LESS file per component.


## Import CSS from relative paths

You may want to import CSS from relative paths like `@JSImport("./MyComponent.css")`, but out of the box, you can't. See [@raquo/vite-plugin-glob-resolver](https://github.com/raquo/vite-plugin-glob-resolver), you can use it together with this plugin.


## Resources
 - https://discord.com/channels/632150470000902164/635668814956068864/1161984220814643241
 - https://discord.com/channels/1020225759610163220/1020225760075718669/1171409017021673575
 - https://vitejs.dev/guide/features.html#disabling-css-injection-into-the-page
 - https://github.com/vitejs/vite/pull/10762
 - https://github.com/vitejs/vite/issues/3246
 - https://github.com/scala-js/scala-js/issues/4156


## Author

Nikita Gazarov – [@raquo](https://twitter.com/raquo)


## License

This project is provided under the MIT license.
