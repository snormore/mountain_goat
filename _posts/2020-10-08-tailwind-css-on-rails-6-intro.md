---
layout: post
title: Getting started with Tailwind CSS on Rails 6
---

If you haven't heard of Tailwind CSS yet, it is a CSS framework filled with a lot of preset CSS classes, which you can apply to your HTML elements, without the need to write custom CSS for them most of the time!



I have experimented with Tailwind CSS in this demo job Rails app : [https://kldev.herokuapp.com](https://kldev.herokuapp.com) , you can view source on the page see that each elements used a lot of CSS classes.



For example this div :

```html
<div class="w-full bg-blue-900">
</div>
```



The **w-full** class :

```css
.w-full {
  width: 100%;
}
```



The **bg-blue-900** class : 

```css
.bg-blue-900 {
  background-color: #2a4365;
}
```



The w-full and bg-blue-900 classes are provided by Tailwind CSS, we can just insert these into the class name to style the element, without needing to open .css file!



## NEW: Tailwind CSS online playground!

You can test out Tailwind CSS online on their playground, by just navigating to https://play.tailwindcss.com on your web browser! Change the class name and HTML on the left editor, and see the realtime preview on the right!



## Trying out without fiddling with javascript

If you would like to try out Tailwind CSS without messing with node.js / npm / yarn, like on a plain .html file, you can use the CDN : 

```html
<!-- Insert this in the head tag -->
<link href="https://unpkg.com/tailwindcss@^1.0/dist/tailwind.min.css" rel="stylesheet">
```



This is great to get a quick demo on Tailwind CSS, but the downside (as mentioned in [official docs](https://tailwindcss.com/docs/installation#using-tailwind-via-cdn)) is that 

- You can't customize Tailwind's default theme
- You can't use any [directives](https://tailwindcss.com/docs/functions-and-directives) like `@apply`, `@variants`, etc.
- You can't enable features like [`group-hover`](https://tailwindcss.com/docs/pseudo-class-variants#group-hover)
- You can't install third-party plugins
- You can't tree-shake unused styles



## Installing Tailwind CSS on your Rails app

I prefer using yarn to install Tailwind CSS, open up terminal, navigate to your Rails app , then execute: 

```bash
yarn add tailwindcss
```



I recommend using Webpacker for managing Tailwind CSS as this will make it easier to manage and we can configure PostCSS setting (like PurgeCSS) later.



To use webpacker for Tailwind CSS, create a folder named **stylesheets** inside **app/javascript** (the full path would be **app/javascript/stylesheets**).



Inside the app/javascript/stylesheets folder, create a file and name it "**application.scss**", then import Tailwind related CSS inside this file :

```css
/* application.scss */
@import "tailwindcss/base";
@import "tailwindcss/components";
@import "tailwindcss/utilities";
```



The file structure looks like this :

![tailwind_import](https://rubyyagi.s3.amazonaws.com/6-tailwind-intro/tailwind_import.png)



Then in **app/javascript/packs/application.js** , require the application.scss : 

```javascript
// app/javascript/packs/application.js

require("@rails/ujs").start()
require("turbolinks").start()

require("stylesheets/application.scss")
```

 



Next, import tailwindcss and autoprefixer plugins in the file **postcss.config.js** (on the root of your Rails app folder) :

```javascript
/* postcss.config.js */
module.exports = {
  plugins: [
    require('tailwindcss'),
    require('autoprefixer'),
    require('postcss-import'),
    require('postcss-flexbugs-fixes'),
    require('postcss-preset-env')({
      autoprefixer: {
        flexbox: 'no-2009'
      },
      stage: 3
    })
  ]
}
```

 



Next, we have to add **stylesheet_pack_tag** that reference the app/javascript/stylesheets/application.scss file in the layout file (app/views/layouts/application.html.erb). So the layout can import the stylesheet there.



```html
<!-- app/views/layouts/application.html.erb-->

<!DOCTYPE html>
<html>
  <head>
    <title>Bootstraper</title>
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>

    <%= stylesheet_link_tag 'application', media: 'all', 'data-turbolinks-track': 'reload' %>
    
    <!-- This refers to app/javascript/stylesheets/application.scss-->
    <%= stylesheet_pack_tag 'application', 'data-turbolinks-track': 'reload' %>

    <%= javascript_pack_tag 'application', 'data-turbolinks-track': 'reload' %>
  </head>
....
```

 



Now we can use the Tailwind CSS classes on our HTML elements!

```html
<h1>This is static#index</h1>
<div class="bg-red-500 w-3/4 mx-auto p-4">
  <p class="text-white text-2xl">Test test test</p>
</div>
```



![tailwind preview](https://rubyyagi.s3.amazonaws.com/6-tailwind-intro/tailwind_preview.png)



You can refer to [Tailwind CSS documentation](https://tailwindcss.com/docs) for the list of classes you can use.



Note: If you notice that the layout / style didn't change even after you have added some classes, try open terminal and run **./bin/webpack-dev-server** alongside with **rails server** so that the styling can be refreshed.



## Changing Tailwind CSS default config

Tailwind CSS comes with a preset configuration, say for the sans-serif fonts it uses this default : 

```css
fontFamily: {
  sans: [
    'system-ui',
    '-apple-system',
    'BlinkMacSystemFont',
    '"Segoe UI"',
    'Roboto',
    '"Helvetica Neue"',
    'Arial',
    '"Noto Sans"',
    'sans-serif',
    '"Apple Color Emoji"',
    '"Segoe UI Emoji"',
    '"Segoe UI Symbol"',
    '"Noto Color Emoji"',
  ]
}
```

 

If you would like to use a different default sans-serif font, you can override the configuration file and insert your own font name.



You can view the full default configuration here : [https://github.com/tailwindlabs/tailwindcss/blob/master/stubs/defaultConfig.stub.js](https://github.com/tailwindlabs/tailwindcss/blob/master/stubs/defaultConfig.stub.js)



You can create a Tailwind configuration file using this command in Terminal (run it on the root of your Rails app) : 

```bash
npx tailwindcss init
```



This will create a minimal `tailwind.config.js` file at the root of your project :

```javascript
// tailwind.config.js
module.exports = {
  future: {},
  purge: [],
  theme: {
    extend: {},
  },
  variants: {},
  plugins: [],
}
```



Any configuration (eg: font-family) you specified in the configuration will override the default which is shown in the Github link above, if you didn't specify the configuration, it will use back the default.



For example, I changed the default font to [Gibson](https://fonts.adobe.com/fonts/gibson) for my demo Rails app, then my **tailwind.config.js** looks like this : 

```javascript
module.exports = {
  future: {
    // removeDeprecatedGapUtilities: true,
    // purgeLayersByDefault: true,
  },
  purge: [],
  theme: {
    fontFamily: {
      sans: ['canada-type-gibson', 'system-ui',
        '-apple-system',
        'BlinkMacSystemFont',
        '"Segoe UI"',
        'Roboto',
        '"Helvetica Neue"',
        'Arial',],
    },
    extend: {},
  },
  variants: {},
  plugins: [],
}

```

 

You can read more on how to customize the default theme configuration here : [https://tailwindcss.com/docs/theme](https://tailwindcss.com/docs/theme)



You can also generate a configuration file with all the default listed inside, although not encouraged by Tailwind themselves : 

```bash
npx tailwindcss init --full
```

 



## Use @Apply to extract component

As you use Tailwind CSS to prototype your design, you might find a lot of HTML elements sharing the same class which you can create a component for it.



For example : 

```html
<div class="bg-red-500 my-4 w-3/4 mx-auto p-4">
  <p class="text-white text-2xl">Test test test 1</p>
</div>

<div class="bg-red-500 my-4 w-3/4 mx-auto p-4">
  <p class="text-white text-2xl">Test test test 2</p>
</div>

<div class="bg-red-500 my-4 w-3/4 mx-auto p-4">
  <p class="text-white text-2xl">Test test test 3</p>
</div>
```

 

We have three divs with the same classes, just different content inside. To avoid typing the same long classes for each of them, we can **extract these classes into one class** by applying the tailwind class on it.



In your **app/javascript/stylesheets/application.scss** file,

```css
/* application.scss */
@import "tailwindcss/base";
@import "tailwindcss/components";
@import "tailwindcss/utilities";


.red-card {
  @apply bg-red-500 my-4 w-3/4 mx-auto p-4;
}

.red-card p {
  @apply text-white text-2xl;
}
```

 



By using @apply, we can apply the attributes in **bg-red-500 my-4 w-3/4 mx-auto p-4** classes to the .red-card custom class we created, we then also apply **text-white text-2xl** to the `<p>` inside red card div.



Now we can refactor our HTML code to this : 

```html
<div class="red-card">
  <p>Test test test 1</p>
</div>

<div class="red-card">
  <p>Test test test 2</p>
</div>

<div class="red-card">
  <p>Test test test 3</p>
</div>
```

 



This makes it easier to reuse the CSS class, instead of having to type all of these classes name again and again.



To avoid unintended specificity issue, Tailwind CSS recommend us to wrap the @apply lines inside the components layer, like this :

```css
/* application.scss */
@import "tailwindcss/base";
@import "tailwindcss/components";
@import "tailwindcss/utilities";

@layer components {
  .red-card {
    @apply bg-red-500 my-4 w-3/4 mx-auto p-4;
  }

  .red-card p {
    @apply text-white text-2xl;
  }
}
```



From Tailwind CSS [documentation](https://tailwindcss.com/docs/extracting-components#extracting-css-components-with-apply) : 

> Tailwind will automatically move those styles to the same place as `@tailwind components`, so you don't have to worry about getting the order right in your source files.
>
> Using the `@layer` directive will also instruct Tailwind to consider those styles for purging when purging the `components` layer.





## PostCSS Purge CSS in production

As Tailwind CSS has **a lot** of classes, and usually we only use a fraction of these classes for styling, it is recommended to purge unused classes from Tailwind CSS to reduce the file size.



From Tailwind CSS documentation : 

> **When building for production, you should always use Tailwind's `purge` option to tree-shake unused styles and optimize your final build size.** When removing unused styles with Tailwind, it's very hard to end up with more than 10kb of compressed CSS.



We can use PurgeCSS plugin for PostCSS to remove unused CSS classes.



Lets install **PurgeCSS** in the terminal : 

```bash
yarn add @fullhuman/postcss-purgecss
```



Then include the purgeCSS plugin in the **postcss.config.js** file (on the root of your Rails app folder) :

```javascript
let environment = {
  plugins: [
    require('tailwindcss'),
    require('autoprefixer'),
    require('postcss-import'),
    require('postcss-flexbugs-fixes'),
    require('postcss-preset-env')({
      autoprefixer: {
        flexbox: 'no-2009'
      },
      stage: 3
    })
  ]
}

// Only run PurgeCSS in production envrionment, you can also add other environment here
if (process.env.RAILS_ENV === "production") {
  environment.plugins.push(
    require('@fullhuman/postcss-purgecss')({
      content: [
        './app/**/*.html.erb',
        './app/**/*.html.haml',
        './app/helpers/**/*.rb',
        './app/javascript/**/*.js',
        './app/javascript/**/*.vue',
        './app/javascript/**/*.jsx',
      ],
      defaultExtractor: content => content.match(/[A-Za-z0-9-_:/]+/g) || []
    })
  )
}

module.exports = environment
```



The script above will push the postcss-purgecss plugin into the plugins array for the environment variable (ie. the `let environment`).



We will explain what action does the script above perform below.



```javascript
content: [
  './app/**/*.html.erb',
  './app/**/*.html.haml',
  './app/helpers/**/*.rb',
  './app/javascript/**/*.js',
  './app/javascript/**/*.vue',
  './app/javascript/**/*.jsx',
],
```



The snippet above means that PurgeCSS script will look for CSS classes name in these files (eg: ./app/views/static/index.html.erb)



For example lets say we have a file in ./**app/views/static/index.html.erb** like this :

```html
<div class="red-card">
  <p>Test test test 2</p>
</div>
```



Then PurgeCSS will scan this file and notice the "red-card" class is in use, and red-card apply a few Tailwind CSS classes like "bg-red-500 my-4 w-3/4 mx-auto p-4",  hence it will not remove these classes.


`'./app/**/*.html.erb'` means that PurgeCSS will scan for any file with **.html.erb** extension, and placed inside the **app** folder or any subfolders inside it.



****** means match any number of subfolders and whatever subfolder name.

**\*** means match any file name.



eg:

app/views/static/index.html.erb matches, 

app/views/sub/sub/sub/bla.html.erb also matches.



Same goes with other extensions mentioned in above snippets (\*.html.haml, app/javascript/\**/*.js etc).



```javascript
defaultExtractor: content => content.match(/[A-Za-z0-9-_:/]+/g)
```



The snippet above means that the extractor (scanner) will scan for CSS class name that contains alphabets (A-Z, a-z), numbers (0-9), dash / hyphens (-) , underscore (_) and colon (:). The "+" means one ore more characters, the "g" means global or greedy match, which matches all the word (ie. CSS classes) that fulfill the pattern (alphabets, numbers, hyphens etc).



### Small gotcha with PurgeCSS

One thing to note with PurgeCSS is that, if the CSS classes your Rails app uses did not appear in the location specified in the **content** array, it will be removed in production.



For example, you might use select2 npm package (`yarn add select2`) for customizing dropdown, and they come with their own css file, which is located in **/node_modules/select2/dist/css/select2.min.css** . As the javascript library will only add the CSS classes to the dropdown on runtime , the PurgeCSS scanner will not detect these classes during scanning, and will remove them on production during deployment.



This will make your dropdown behave weirdly on production as the select2 CSS classes are not applied on runtime. To solve this, we can use **safelist**, which tell PurgeCSS to not remove classes that match certain regular expression.



For the select2 example, I want PurgeCSS to allow CSS classes that contain "select" to exist (not removed), then I add this into the safelist like this :

```javascript
// Only run PurgeCSS in production (you can also add staging here)
if (process.env.RAILS_ENV === "production") {
  environment.plugins.push(
    require('@fullhuman/postcss-purgecss')({
      content: [
        './app/**/*.html.erb',
        './app/helpers/**/*.rb',
        './app/javascript/**/*.js',
        './app/javascript/**/*.vue',
        './app/javascript/**/*.jsx',
      ],
      safelist: {
        greedy: [/select/],
      },
      defaultExtractor: content => content.match(/[A-Za-z0-9-_:/]+/g) || []
    })
  )
}
```



The **greedy** will select any selectors that matches the regular expression in any part, eg : **button.select2-abc.nonexistent-class** will match the /select/ .



There's also **standard** and **deep** options for the safelist, you can read more in the [PurgeCSS safelisting documentation here](https://purgecss.com/safelisting.html#specific-selectors).

<script async data-uid="d862c2871b" src="https://rubyyagi.ck.page/d862c2871b/index.js"></script>