# Scala.js Import Side Effect Vite Plugin

This [Vite](https://vitejs.dev/) plugin lets you import resources for their side effects only, without capturing their value. Basically, it lets Scala.js make plain imports like `import "foo.css"` instead of `import * as unused from "foo.css"`, which is what JSImport does.

This is especially relevant to importing CSS resources, because if we do `JSImport("foo.css")`, which results in `import * as unused from "foo.css"`, Vite production build will include the resulting CSS into the JS bundle, but it will also generate a CSS file containing this CSS. So, CSS will be double-loaded, which is wasteful. It's also annoying to see duplicate CSS declarations in browser dev tools.

On the other hand, if we said `import "foo.css"`, Vite production build would only include the CSS in the CSS file, it would not add it to the JS bundle.

At the moment, Scala.js can't emit `import "foo.css"`, so this plugin does _unspeakable things_ to simulate that. **This plugin is a stopgap / proof of concept, there are ways to make it better – read on.**


## Usage

Add this Scala file to your Scala.js project:

```scala
object JSImportSideEffect {

  @js.native
  @JSGlobal("importSideEffect_3DfPjKW0ZYyY")
  def apply(moduleId: String): Unit = js.native
}
```

Install this plugin:

```
npm i -D @raquo/vite-plugin-import-side-effect
```

Add this plugin to your vite.config.js (IN THIS ORDER, if used together):

```js
import importSideEffectPlugin from "@raquo/vite-plugin-import-side-effect";

// Add to the list of plugins:
importSideEffectPlugin({
  importFnName: "importSideEffect_3DfPjKW0ZYyY"
})
```

Also, install my [glob-resolver]("https://github.com/raquo/vite-plugin-glob-resolver") plugin, unless you want to manually specify long paths.

**Make sure to read the caveats below.**


## Unspeakable Things

When you call `JSImportSideEffect("foo.css")` in your Scala.js code, Scala.js outputs the following JS code: `importSideEffect_3DfPjKW0ZYyY("foo.css")`.

The plugin works by finding `importSideEffect_3DfPjKW0ZYyY("foo.css")` in your JS code, commenting it out by prepending `// `, and putting an `import "foo.css"` import statement at the top of the module, prior to any other Scala.js-produced import statements. This is done using [magic-string](https://github.com/rich-harris/magic-string#readme), so the source maps remain valid.

Therefore, our `JSGlobal` name must match the `importFnName`, and must be a unique string in your codebase. You can provide your own, if you don't like `importSideEffect_3DfPjKW0ZYyY` for some reason.

So, _everything is fine_. 


### Caveats

* Avoid injecting untrusted code into your JS files, or generating untrusted JS code. This plugin will find and replace `importSideEffect_3DfPjKW0ZYyY("foo")` in your .js files, **even if that pattern is found in string literals or comments**. (Note: Comments in Scala files are not output to JS). Beware of code generation that works on untrusted input and outputs .scala or .js files, including sbt-buildinfo plugin if it outputs any untrusted data. Failure to do that can allow an attacker to inject an `import "foo"` statement into your JS code.

  * **This can be improved – see TODO section below. This PoC is working well for me, but PRs to help with those issues are welcome.**

  * For now, if this is a real concern to you, I recommend changing the `importSideEffect_3DfPjKW0ZYyY` string to something different.

* Put `importSideEffectPlugin` prior to any other plugin that could alter Scala.js output in a way that could make our transformations unsafe.

* The plugin isn't suitable for library authors who need to import CSS files, only for end users building their applications. But, this can be improved.


## TODO

* An alternative, potentially less horrifying, way to achieve the desired effect is to replace Scala.js-generated `import * as unused from "foo.css"` imports with `import "foo.css"`, however this `unused` is a random identifier, and I don't know how to tell which of them are actually unused in the code. Can we just check for the occurrences of those `unused` strings in the same module, and if not found, assume that they're not used?

* Currently we allow to rewrite any imports. Since this plugin is really only beneficial for CSS files, perhaps we should limit the imports to `*.css` by default, prevent URL imports (are those allowed in Vite?), etc.  

* Currently we apply this transformation to all `.js` files that aren't in `node_modules`. We only need to apply it to files produced by scala.js, but I'm not sure how to tell which ones those are.

* I'm hoping that the need for this plugin will eventually be eliminated one way or another.


## Related
 - https://discord.com/channels/632150470000902164/635668814956068864/1161984220814643241
 - https://vitejs.dev/guide/features.html#disabling-css-injection-into-the-page
 - https://github.com/vitejs/vite/pull/10762
 - https://github.com/vitejs/vite/issues/3246
 - https://github.com/scala-js/scala-js/issues/4156


## Author

Nikita Gazarov – [@raquo](https://twitter.com/raquo)


## License

This project is provided under the MIT license.
