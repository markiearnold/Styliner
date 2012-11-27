﻿#Styliner
Styliner is a Node.js library that reads CSS rules from external stylesheets and converts them to inline `style=""` attributes in an HTML document.  

Styliner is intended for use with HTML emails.  With it, you can write regular CSS or LESS (with the Styliner-less package) stylesheets, then merge them into your HTML and create emails that work with Gmail (which drops all `<style>` tags).  Unfortunately, though, you'll still need to use `<table>`s to get complex layout.

You can also use advanced features ("dynamic rules") like `:hover` selectors or media queries, and Styliner will leave them in a `<style>` tag.  This way, you can build interactive emails that will light up when viewed in an email client that supports `<style>` tags, while still maintaining Gmail support.  

In effect, you get graceful degradation for your email designs.

##Usage
Styliner uses the [Q](https://github.com/kriskowal/q) promise library.

```javascript
var styliner = new Styliner(baseDir, { options });
styliner.processHTML(htmlSource, directory)
    .then(function(source) { ... });
```

The `baseDir` parameter specifies the base directory for relative paths.  When processing an HTML source file, you can optionally specify a directory for that source, and any relative paths within the file will be treated as relative to that directory (instead of relative to the `Styliner` instance's `baseDir`).  
The `processHTML` method returns a Q promise of the inlined HTML source.

You can pass an options hash as the second parameter to the `Styliner` constructor with the following options: (all options default to false):

 - `compact: true`		
  - True to minify all output.  This option removes all extraneous whitespace from the generated HTML (including any remaining inline stylesheets)   
 - `noCSS: true`
  - True to not emit `<style>` tags for rules that cannot be inlined.  This option will completely drop any dynamic CSS rules. (such as media queries, pseudo-elements, or `@font-face`s)
 - `keepRules: true`
  - True to keep all rules in `<style>` tags instead of inlining static rules into elements.  This results in smaller files, but will not work with Gmail.
 - `fixYahooMQ: true`
  - True to add an attribute/ID selector to all rules in media queries to fix a bug in Yahoo Mail.  Yahoo Mail drops all media queries, converting their contents to regular rules that will always be applied.  This option adds a workaround for that.
 - `keepInvalid: true`
  - Don't skip properties that parserlib reports as invalid. (all invalid properties are always logged as winston errors)
  - Pass this option if you're using features that parser doesn't recognize, or if you come across a bug in parserlib
  - This option breaks Acid2, which tests that valid properties from earlier rules replace invalid properties from later rules.  (see also the first known issue)
 - `urlPrefix: "dir/"`
  - The path containing referenced URLs.  All non-absolute URLs in `<a>` tags, `<img>` tags, and stylesheets will have this path prepended.  For greater flexibility, pass a `url()` function instead.
 - `url: function(path, type)`
  - A function called to resolve URLs.  All non-absolute URLs in HTML or CSS will be replaced by the return value of this function. The function is passed the relative path to the file and the source of the URL ("img" or "a" or other HTML tags; URLs from CSS pass "img"). It can return a promise or a string

##Known Issues
 - Browser property fallbacks don't cascade
   - If you specify `background: red;` in one rule, and `background: linear-gradient(...)` in a more specific rule, Styliner will replace the property from the first rule with the more specific one.  This means that browsers that don't support `linear-gradient()` won't see any background at all. 
   - Instead, put both properties in the same rule, and Styliner will know to keep both of them.  To make this easier, you can use a LESS mixin
 - Except for `margin` and `padding`, shorthand inlined properties that are overridden by non-inlined non-shorthand counterparts will not be overridden correctly.
  - To fix this, add splitter methods in Preprocessor.js to split other shorthand properties.

  ###CSSselect issues
  The Acid2 and Acid3 tests do not work when run through Styliner because the [CSSselect](https://github.com/fb55/CSSselect) parser (which I use to find elements to apply styles to) cannot handle exotic selectors.

  Specifically, the following selectors don't work:
   - `[class~="one"][class~="first"] [class="second two"][class="second two"]` (Acid2) doesn't match correctly
   - `* html .parser` incorrectly matches `.parser`; this may be my fault
   - `#\ ` (an escaped ID selector matching `id=" "`) crashes the CSSselect parser for Acid3 (https://github.com/fb55/CSSwhat/issues/2)