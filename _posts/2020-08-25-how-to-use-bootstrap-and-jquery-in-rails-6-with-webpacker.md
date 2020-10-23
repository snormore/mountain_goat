---
layout: post
title: How to use Bootstrap, jQuery and other libraries in Rails 6 with Webpacker
---


Since Rails 6 , Webpacker has replaced the old assets pipeline (sprockets) to handle the javascript compilation and minification.



Webpacker is a gem which is a wrapper for [webpack.js](https://webpack.js.org), webpack.js handles bundling of javascript code, and webpacker lets us interface with webpack in our Rails app. This article won't go deep on how webpacker works, I recommend [Prathamesh's article on understanding webpacker](https://prathamesh.tech/2019/08/26/understanding-webpacker-in-rails-6/) if you want to know more on how webpacker works. GoRails also have a [great screencast](https://gorails.com/episodes/how-to-use-bootstrap-with-webpack-and-rails) on this topic.



This article assumes you are starting a new Rails 6 project and want to use Bootstrap.



1. [Installing and using Bootstrap and jQuery](#installing)
2. [Using jQuery in view code](#view)
3. [Using a library that requires jQuery](#library)
4. [Using custom javascript library which is not available on npm](#custom)



<span id="installing"></span>

## Installing and using Bootstrap and jQuery


First, open terminal and type the command below to install Bootstrap, jQuery and popper.js (required for some bootstrap components) :



```bash
yarn add bootstrap jquery popper.js
```


 

Next, we will create a folder named **stylesheets** inside **app/javascript** (the full path would be **app/javascript/stylesheets**), this would behave similar like app/assets/stylesheets, except that the stylesheet files here are usually bundled with the javascript module. (eg: the bootstrap module bundles its CSS together).



Inside the app/javascript/stylesheets folder, create a file and name it "**application.scss**" (similar to assets). Then import the bootstrap CSS (from the javascript module) in this file : 

```css
/* app/javascript/stylesheets/application.scss */
@import "~bootstrap/scss/bootstrap";
/* this will import the scss file inside node_modules/bootstrap/scss/bootstrap.scss
```

 

*Optional step* - I understand that you might not feel comfortable placing a stylesheet file inside the javascript folder. If you prefer to organize all stylesheet in the assets folder, you can import the bootstrap in app/assets/stylesheets/application.scss (rename it to **.scss if it is still .css**) instead.

```javascript
/* app/assets/stylesheets/application.scss */

@import '../../../node_modules/bootstrap/scss/bootstrap';
```

 

Then in **app/javascript/packs/application.js** , import bootstrap and its bundled CSS, so that webpack will load these dependency : 

```javascript
// app/javascript/packs/application.js

require("@rails/ujs").start()
require("turbolinks").start()
require("@rails/activestorage").start()
require("channels")

// import the bootstrap javascript module
import "bootstrap"

// import the application.scss we created for the bootstrap CSS (if you are not using assets stylesheet)
import "../stylesheets/application"
```

 

![import css](https://rubyyagi.s3.amazonaws.com/3a-bootstrap-jquery-rails-6/import_css.png)



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

 

As Bootstrap requires jQuery and popper.js to function, we would need to use ProvidePlugin to make jQuery and popper.js modules to be available to other modules (ie. bootstrap in this case).



In modern modularized javascript development, modules usually can't access variable inside other module unless you import them manually, this is to encourage writing isolated modules that are well contained and do not rely on hidden dependencies (e.g. globals).

![isolated modules](https://rubyyagi.s3.amazonaws.com/3a-bootstrap-jquery-rails-6/provide1.png)



By using ProvidePlugin to perform [shimming](https://webpack.js.org/guides/shimming/), we can 'provide' the variable to other module.

> The [`ProvidePlugin`](https://webpack.js.org/plugins/provide-plugin) makes a package available as a variable in every module compiled through webpack. If webpack sees that variable used, it will include the given package in the final bundle.

![provide plugin](https://rubyyagi.s3.amazonaws.com/3a-bootstrap-jquery-rails-6/provide2.png)



Open **config/webpack/environment.js** , and then add the provide plugin, and provide "**$**", "**jQuery**" and "**Popper**" variables.



```javascript
// config/webpack/environment.js
const { environment } = require('@rails/webpacker')
const webpack = require("webpack");

// Add an additional plugin of your choosing : ProvidePlugin
environment.plugins.append(
  "Provide",
  new webpack.ProvidePlugin({
    $: "jquery",
    jQuery: "jquery",
    Popper: ["popper.js", "default"] // for Bootstrap 4
  })
);

module.exports = environment
```

 

You might need to restart your rails server for the change in **environment.js** to update.



And now you should be able to use Bootstrap on your view code!


<span id="view"></span>
## Using jQuery in view code

At this point if you try to use jQuery in your layout code (eg: html.erb or html.haml), you would get an error "Uncaught ReferenceError, $ is not defined"  :

![uncaught reference error](https://rubyyagi.s3.amazonaws.com/3a-bootstrap-jquery-rails-6/uncaught_reference.png)



This is because we are attempting to use "$" on a global scope, but "$" is not available on the global scope. The ProvidePlugin only makes it available to other modules, but not on global.

![not on global](https://rubyyagi.s3.amazonaws.com/3a-bootstrap-jquery-rails-6/no_access.png)



To solve this, we can attach the "$" and "jQuery" variable to the global scope (or window scope as this is the top most scope used in web browsers). In the pack file (**app/javascript/packs/application.js**), attach them to global : 



```javascript
// app/javascript/packs/application.js

require("@rails/ujs").start()
require("turbolinks").start()
require("@rails/activestorage").start()
require("channels")

import "bootstrap"
import "../stylesheets/application"

var jQuery = require('jquery')

// include jQuery in global and window scope (so you can access it globally)
// in your web browser, when you type $('.div'), it is actually refering to global.$('.div')
global.$ = global.jQuery = jQuery;
window.$ = window.jQuery = jQuery;
```

 

<span id="library"></span>
## Using a library that requires jQuery

For this example, I will use the library [DateRangePicker](https://www.daterangepicker.com) , as I have used this for date selection in many web apps. This library requires jQuery and moment.js to function.



First, install moment.js and daterangepicker using yarn as they are available on yarn : 

```bash
yarn add moment daterangepicker
```

 


Next we will include moment in the global scope as DateRangePicker might require it, open up the pack file **app/javascript/packs/application.js**, add require moment and bind it to the global scope.



Don't forget to import daterangepicker, and make sure it is imported **after** bootstrap and jQuery.

```javascript
// app/javascript/packs/application.js

require("@rails/ujs").start()
require("turbolinks").start()
require("@rails/activestorage").start()
require("channels")

import "bootstrap"
import "../stylesheets/application"

var moment = require('moment')

var jQuery = require('jquery')

import "daterangepicker"

// include jQuery in global and window scope (so you can access it globally)
// in your web browser, when you type $('.div'), it is actually refering to global.$('.div')
global.$ = global.jQuery = jQuery;
window.$ = window.jQuery = jQuery;

// include moment in global and window scope (so you can access it globally)
global.moment = moment;
window.moment = moment;
```

 


As DateRangePicker comes with its own css, we need to import it in the stylesheet pack file. (**app/javascript/stylesheets/application.scss**)

```css
/* app/javascript/stylesheets/application.scss */

@import "~bootstrap/scss/bootstrap";

@import "../../../node_modules/daterangepicker/daterangepicker.css";
```

 

For some reason, using "~" doesn't work on .css file, so we can't use @import "~daterangepicker/daterangepicker.css". So we have to resort to navigating up three folders from /app/javascript/stylesheets to /node_modules using "../../../node_modules".



 If the node module has .scss file, you can import with the "~" symbol in front, as it means the path to the node_modules folder.



Now in your view code, you can set an input to be daterangepicker like this : 

```html
<h1>Page#index</h1>
<p>Find me in app/views/page/index.html.erb</p>
 
<input type="text" name="dates"/>
<script>
$( document ).ready(function() {
  $('input[name="dates"]').daterangepicker();
});
</script>
```

 

<span id="custom"></span>
## Using custom javascript library which is not available on npm

What if the javascript library you want to use isn't available on NPM? There's some libraries that only available as a standalone .js file, how do you install and use this with webpacker?



For this example I am going to use the [select2](https://select2.org/getting-started/installation) library, it has a gem ([select2-rails](https://github.com/argerim/select2-rails)), but we are going to download its .js and .css and include it in our Rails app. We can download the release code here : https://github.com/select2/select2/tags , select a version, unzip it and go to the "dist" folder, then  find **select2.min.css** and **select2.min.js**.



Usually I will create a folder for custom javascript library and place it inside the **app/javascript** folder, in this case, **app/javascript/select2**, and place the css and js file inside like this : 

![js css location](https://rubyyagi.s3.amazonaws.com/3a-bootstrap-jquery-rails-6/custom_js.png)



Then in your application pack file (**app/javascript/packs/application.js**), import the select2.min.js file like this : 

```javascript
// app/javascript/packs/application.js

import "../select2/select2.min"
```

 

We can omit the ".js" file extension, make sure the path is correct.



Next, we need to import css file in the stylesheet pack file we created earlier (**app/javascript/stylesheets/application.scss**)

```css
/* app/javascript/stylesheets/application.scss */
@import "../select2/select2.min.css"
```

 

And now we can use the select2 functionality in view code : 

```html
<select class="js-example-basic-single" name="state">
  <option value="AL">Alabama</option>
  <option value="WY">Wyoming</option>
</select>
 

<script>
$( document ).ready(function() {
  $('.js-example-basic-single').select2();
});
</script>
```

 <script async data-uid="4776ba93ea" src="https://rubyyagi.ck.page/4776ba93ea/index.js"></script>



