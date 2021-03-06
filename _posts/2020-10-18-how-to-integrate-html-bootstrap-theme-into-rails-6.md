---
layout: post
title: How to integrate HTML / Bootstrap template into Rails 6
---

Say you just bought a Bootstrap template (or any HTML template) and want to integrate it to Rails 6, where do you start? 🤔



This tutorial aims to guide the integration process, outlining a few main steps, the whole integration process might take a few hours to a few days depending on how big or complex for the template you bought.

**Table of contents**
1. [Install related NPM packages](#install-related-npm-packages)
2. [Bootstrap, Jquery and Popper.js](#bootstrap-jquery-and-popperjs)
3. [Install custom (proprietary) Javascript libraries](#install-custom-proprietary-javascript-libraries)
4. [Copying CSS and Images](#copying-css-and-images)
5. [Copying HTML Files](#copying-the-html-files)
6. [Debugging Javascript issues](#debugging-javascript-issues)


I will be using the [Front Bootstrap Theme](https://themes.getbootstrap.com/product/front-multipurpose-responsive-template/) for this tutorial, you can use your own theme, I think the steps should be similar.



![front bootstrap screenshot](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/screenshot.png)



I suggest creating a controller and a view, and route the root path to this view before continuing if you are creating a new Rails app. As the default 'Yay you are on Rails' page is not using the NPM packages which we are gonna install the next part, hence you won't see any error if anything goes wrong, which we don't want as we want to be able to resolve potential javascript issues.



## Install related NPM packages

Many templates rely on NPM packages for frontend stuff, we can check which NPM packages they are using   by navigating to the **javascript** or **vendor** folder of the template : 

![vendor](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/vendorjs.png)





One thing to note is that not all Javascript libraries used by the template are available as NPM package, some of the Javascript library might be proprietary to the template, or just not available on NPM.



You can check if the Javascript library is available as package by searching it on the NPM website : [https://www.npmjs.com](https://www.npmjs.com)



List down the names of the libraries that are available on NPM, then install them all using `yarn add` in terminal, for my example : 



`yarn add @yaireo/tagify appear bootstrap popper.js chart.js chartjs-chart-matrix chartjs-plugin-datalabels circles.js clipboard datatables datatables.net-buttons daterangepicker dropzone flag-icon-css flatpickr ion-rangeslider jquery-mask-plugin jquery-migrate jquery-validation jszip leaflet list.js pdfmake pwstrength-bootstrap quill sortablejs table-edits `



(Yes, thats a lot) If you are using bootstrap package, make sure to install "popper.js" package as well, as some bootstrap components relies on popper.js .



Now that we have installed these NPM packages, lets import them in the main pack file in **app/javascript/packs/application.js** 



```javascript
/* app/javascript/packs/application.js */
// This file is automatically compiled by Webpack, along with any other files
// present in this directory. You're encouraged to place your actual application logic in
// a relevant structure within app/javascript and only use these pack files to reference
// that code so it'll be compiled.

require("@rails/ujs").start()
require("turbolinks").start()


// Uncomment to copy all static images under ../images to the output folder and reference
// them with the image_pack_tag helper in views (e.g <%= image_pack_tag 'rails.png' %>)
// or the `imagePath` JavaScript helper below.
//
// const images = require.context('../images', true)
// const imagePath = (name) => images(name, true)

import "@yaireo/tagify";
import "appear";
import "bootstrap";
import "chart.js";
import "chartjs-chart-matrix";
import "chartjs-plugin-datalabels";
import "circles.js";
import "clipboard";
import "datatables";
import "datatables.net-buttons";
import "daterangepicker";
import "daterangepicker/daterangepicker.css";
import "dropzone";
import "flag-icon-css/css/flag-icon.min.css"
import "flatpickr";
import "flatpickr/dist/flatpickr.min.css";
import "ion-rangeslider";
import "jquery-mask-plugin";
import "jquery-migrate";
import "jquery-validation";
import "jszip";
import "leaflet";
import "list.js";
import "pdfmake";
import "pwstrength-bootstrap/dist/pwstrength-bootstrap.min";
import "quill";
import "select2";
import "select2/dist/css/select2.min.css";
import "sortablejs";

```

 

Alright now we have imported these NPM packages, lets run **rails server** and **webpack dev server** to check if everything is working alright : 

```bash
rails server
```

 

then open a new terminal tab/window : 

```bash
./bin/webpack-dev-server
```

 

Head over to http://localhost:3000 , oh no theres an error!

![module not found error](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/fail_to_compile.png)



What does the "*Module not found: Error Can't resolve 'appear'* " error mean? It means that the **import 'appear';** line in the application.js has failed, as there is no 'appear' module to be imported.



Usually modern Javascript package will export their functionality as a module, so that we can import them easily. Say for example the **@yaireo/tagify** library, we can navigate to **node_modules/@yaireo/tagify** folder, and open the **package.json** file : 

![package.json of tagify](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/main_from_package.png)



There's a "**main**" key which tell us which is the main file of this library, in this case its "**./dist/tagify.min.js**", let's open it.



If we search the keyword "export" in the **tagify.min.js** file, there should be at least one match :

![export search](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/export_module.png)



The '**module.exports**' in the Javascript code will export the functionality of the library as a module, most modern Javascript library will have this line.



Now let's look back at the problematic '**appear**' package, navigate to **node_modules/appear/package.json** and open it : 

![appear package.json](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/main_from_appear.png)



The "main" key refered to "**appear.js**", which is actually located in **dist/appear.js**. Let's open up this file and search for the keyword "export" :

![no export found](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/unable_to_find_export.png)



Turns out there's no export keyword in this library! As its functionality is not exported, we can't import it, this is why "**import 'appear';**" line doesn't work.



To solve this, we can use the "require" statement instead of import, and usually the file to be required is mentioned in the "main" key in package.json, which in this case, the "appear.js" file.



In your **app/javascript/packs/application.js** file : 

```javascript
import "@yaireo/tagify";
// change the import to require
// import "appear";
require("appear/dist/appear");
import "bootstrap";
import "chart.js";
```

 



Note: For javascript file, we can skip the ".js" extension, `require("appear/dist/appear")` is equivalent to `require("appear/dist/appear.js")`. But for CSS, we must add the ".css" extension.



Save it, and refresh your page in web browser, it works now!



If you find yourself facing the 'Module not found' error after installing NPM package and importing them, you can try change the "import" into "require" for the problematic package.



There might also be package that doesn't have "main" key specified in their package.json file, in this case, you have to search for the library's javascript file path, and import or require the full path like this : 

```javascript
// there's no "main" key specified in package.json, you have to import the full path to the actual javascript file
// import pwstrength-bootstrap
import "pwstrength-bootstrap/dist/pwstrength-bootstrap.min";

// there's no "main" key specified in package.json, and the javascript file doesn't export as module, we have to use require
require("table-edits/build/table-edits.min");

```

 

In some case, the imported library is actually just CSS, you have to import the full path to the CSS file : 

```javascript
import "flag-icon-css/css/flag-icon.min.css"
```

 



There might also have some Javascript package that comes with their own CSS file, we have to import these as well so that the styling will apply. For example, the package **daterangepicker**, **flatpickr** and **select2** comes with their own CSS files, and we need to import them too  :

```javascript
/* app/javascript/packs/application.js */

// ...
import "daterangepicker";
import "daterangepicker/daterangepicker.css";

import "flatpickr";
import "flatpickr/dist/flatpickr.min.css";

import "select2";
import "select2/dist/css/select2.min.css";
```

 

"*How do I know which library has their own CSS files?*" , I heard you, I also asked the same question, I am no Javascript expert, I only realize I need to import these CSS when I saw the dropdown on the HTML page behave weirdly, I don't have easy answer for this.



For [daterangepicker](http://www.daterangepicker.com) , their getting started section has a stylesheet include, hence I figured I need to import CSS in the application.js as well.

![getting started](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/include_style.png)



If your Javascript components behave or look weird, chances are they come with their own CSS and you need to import them too.

 

## Bootstrap, Jquery and Popper.js

As I have covered in [this guide on how to use Bootstrap and Jquery on Rails 6](https://rubyyagi.com/how-to-use-bootstrap-and-jquery-in-rails-6-with-webpacker/), we need to use ProvidePlugin to perform shimming so modules can use variables in other modules.



Open **config/webpack/environment.js** , and add Jquery, Bootstrap and Popper to the ProvidePlugin :

```javascript
const { environment } = require('@rails/webpacker')
const webpack = require("webpack");

// Add an additional plugin of your choosing : ProvidePlugin
environment.plugins.prepend(
  "Provide",
  new webpack.ProvidePlugin({
    $: "jquery",
    jQuery: "jquery",
    Popper: ["popper.js", "default"] // for Bootstrap 4
  })
);

environment.loaders.get('sass').use.splice(-1, 0, {
  loader: 'resolve-url-loader'
})

module.exports = environment
```

 



## Install custom (proprietary) Javascript libraries

For custom libraries which is not available on NPM, usually I will create a folder named "vendor" inside app/javascript, which makes the full path **app/javascript/vendor**.



Copy paste the custom libraries into the **app/javascript/vendor** folder : 

![vendor jses](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/vendor_jses.png)



Then import or require these libraries in **app/javascript/packs/application.js** , use the full path to the actual Javascript file. 



Use 'import' if the library has exported itself as a module (search "export" in the library's javascript file, if not found, then use require), or use 'require' if the library didn't export itself as module.



Some libraries might have their own CSS files, remember to import those CSS files as well.



Here's a short example of importing those vendor Javascript files :

```javascript
// app/javascript/packs/application.js

// ....

import "../vendor/babel-polyfill/polyfill.min"
import "../vendor/chart.js.extensions/chartjs-extensions"

import "../vendor/hs-add-field/dist/hs-add-field.min"
import "../vendor/hs-count-characters/dist/js/hs-count-characters"
import "../vendor/hs-counter/dist/hs-counter.min"
import "../vendor/hs-file-attach/dist/hs-file-attach.min"
import "../vendor/hs-form-search/dist/hs-form-search.min"
import "../vendor/hs-fullscreen/dist/hs-fullscreen.min.css"
import "../vendor/hs-fullscreen/dist/hs-fullscreen.min"
import "../vendor/hs-go-to/dist/hs-go-to.min"
import "../vendor/hs-loading-state/dist/hs-loading-state.min"
import "../vendor/hs-loading-state/dist/hs-loading-state.min.css"
import "../vendor/hs-mega-menu/dist/hs-mega-menu.min.css"
import "../vendor/hs-mega-menu/dist/hs-mega-menu.min"

// ...
```

 

The "**../**" means go up one level from packs folder, to the "javascript" folder, then "**../vendor**" will access vendor folder from the "javascript" folder.



If you are facing "module not found" error, remember to check if the library's javascript code has "export" keyword in it. If there's no "export" keyword in it, try use "require" for the library.



There might also be theme-specific javscript files, I usually place them under **app/javascript/src** folder like this : 



![theme specific javascript](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/themejs.png)



Then import them as well inside **app/javascript/packs/application.js** file : 

```javascript
// app/javascript/packs/application.js

// ...

import "../src/theme-custom";
import "../src/theme.min";
import "../src/hs.chartjs-matrix"
```

 



We are done with javascript libraries for now, next we will look into copying CSS files and images.



## Copying CSS and Images

This tutorial will be using Webpack for handling CSS and images, if you prefer using Asset Pipeline, thats fine too, but it will not be covered by this tutorial.



Copy the CSS and images folder into app/javascript folder like this : 

![theme css](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/theme_css.png)



Then create a '**application.scss**' file inside **app/javascript/css** folder (or whatever your CSS folder is named, could be 'stylesheets' etc).



And inside the **application.scss** file, import all the CSS files inside the css folder : 

```css
/* app/javascript/css/application.scss */

@import "docs.min";
@import "theme.min";
```

 



Next, head to **app/javascript/packs/application.js** , and import the application.scss file and also the img and svg folder :

```javascript
/* app/javascript/packs/application.js */

// import the application.scss we created for the CSS
import "../css/application.scss";

// copy all static images under ../img and ../svg to the output folder,
// and you can reference them with <%= image_pack_tag 'media/img/abc.png' %> or <%= image_pack_tag 'media/svg/def.svg' %>
const images = require.context('../img', true)
const imagePath = (name) => images(name, true)

const svgs = require.context('../svg', true)
const svgPath = (name) => svgs(name, true)

// ...

```

 



The code above will allow you to reference and load the application.scss using **stylesheet_pack_tag** , and use the images using **image_pack_tag**.



Next, head to **app/views/layouts/application.html.erb**, and include the stylesheet pack tag so the custom CSS files will be loaded for all layout : 

```html
<!DOCTYPE html>
<html>
  <head>
    <title>PaidTemplate</title>
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>

    <%= stylesheet_link_tag 'application', media: 'all', 'data-turbolinks-track': 'reload' %>
    
    <!-- This will load the app/javascript/css/application.scss and its imported CSS files -->
    <%= stylesheet_pack_tag 'application', 'data-turbolinks-track': 'reload' %>

    <%= javascript_pack_tag 'application', 'data-turbolinks-track': 'reload' %>
  </head>

  <body>
    <%= yield %>
  </body>
</html>
```

 



Next, we will move to the HTML files.



## Copying the HTML files

For this tutorial, I will be copying only one HTML file, but the steps should be similar for all HTML files in your theme.



I picked "apps-invoice-generator.html" page from the theme I purchased, I have created a new view named **invoice_generator.html.erb** and placed it under the 'static' controller I have created.



Copy paste everything between the `<body>` and `</body>`  tag of the template HTML to this view.



Run `rails s` and `./bin/webpack-dev-server` , and navigate to localhost to view your page! 



p/s: If you saw an error saying **module not found: Error: Can't resolve 'resolve-url-loader'**, you can solve it by simply installing this package : 

```bash
yarn add resolve-url-loader
```



On first glance, the page looks nice!

![first glance](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/first_glance.png)



Just the some images are missing, as the original theme used relative path to the image folder, we have to change them to use the image_pack_tag.



For example, this svg logo image tag : 

```html
<img class="navbar-brand-logo" src="./assets/svg/logos/logo.svg" alt="Logo">
```

 

We have to change it into this :

```html
<%= image_pack_tag "media/svg/logos/logo.svg", class: 'navbar-brand-logo', alt: 'Logo' %>
```

 





Let's say an image is located in **app/javascript/img/160x160/img2.jpg**

![Image path](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/img_path.png)



We can then replace the "app/javascript" with "**media**" , for the image_pack_tag :

```html
<%= image_pack_tag "media/img/160x160/img2.jpg", class: 'avatar avatar-xs avatar-circle mr-2', alt: 'avatar' %>
```

 



After replacing all the image tags, you are almost set!



## Debugging Javascript issues

As I click around, the dropdown menu doesn't work, and the slide menu doesn't work either, maybe there's some Javascript issue I missed?



Right click the web page, and select "Inspect element" (or "Inspect" if you are using Chrome), then navigate to the Console tab , there's a "**Uncaught ReferenceError: $ is not defined**" error!



![inspect element](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/inspect.png)



![js error](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/js_error.png)



This is because the HTML contains `<script>` tags that uses `$(...)` Jquery function directly, which the "$" is not available on the global scope. ([You can read more about this here](https://rubyyagi.com/how-to-use-bootstrap-and-jquery-in-rails-6-with-webpacker/)).



To solve this, we can attach the "$" and "jQuery" variable to the global scope (or window scope as this is the top most scope used in web browsers). In the pack file (**app/javascript/packs/application.js**), attach them to global :



```javascript
// app/javascript/packs/application.js

// ...

// export jquery to global
var jQuery = require('jquery')

// include jQuery in global and window scope (so you can access it globally)
// in your web browser, when you type $('.div'), it is actually refering to global.$('.div')
global.$ = global.jQuery = jQuery;
window.$ = window.jQuery = jQuery;
```

 



Refresh the page again, and now the  "**Uncaught ReferenceError: $ is not defined**" error is gone!  



Aside from `$` , you might also encounter other **Uncaught ReferenceError** when some proprietary libraries' classes are used in the view code, for example : 

![proprietary error](https://rubyyagi.s3.amazonaws.com/9-integrate-bootstrap-template/proprietary_error.png)



The solution is same as above, we just need to attach the 'HSUnfold' variable (replace this with the variable name you saw) to the global or window scope, then the view code can access it.

```javascript
// app/javascript/packs/application.js
// ...

var unfold = require('../vendor/hs-unfold/dist/hs-unfold.min.js')
global.HSUnfold = unfold;
window.HSUnfold = unfold;
```

 

Repeat this process until all the variables used in the view code is exposed on the global scope.



After all this, you have successfully integrated your theme into the Rails app! 🥳 (Although just one page, all the best for integrating the rest of it 😂)



<script async data-uid="4776ba93ea" src="https://rubyyagi.ck.page/4776ba93ea/index.js"></script>

