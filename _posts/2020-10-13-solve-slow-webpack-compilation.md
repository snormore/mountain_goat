---
layout: post
title: Solve slow webpack compilation when using Tailwind @apply
---

In the [introduction to Tailwind post](https://rubyyagi.com/tailwind-css-on-rails-6-intro/), I mentioned using @apply to extract components like this  :

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

<br>

Using this worked fine for me, but I started noticing the page would take a while to refresh whenever I made change on the **application.scss** file, even with just two @apply statements, it took nearly **5 seconds** to recompile on webpack! 

![slow](https://rubyyagi.s3.amazonaws.com/8-solve-slow-compile-tailwind/slow.png)

Means every time I made change on the CSS, I have to wait 5 seconds to see the output, and multiply it with numbers of time I make changes on the CSS, the feedback loop becomes really slow 😨! 



It seems like an issue related to webpack and the heavy file size of Tailwind's utilities CSS classes, as mentioned in this [Github issue](https://github.com/tailwindlabs/tailwindcss/issues/443).



I don't have a thorough understanding on why this issue happen, but my guess is that webpack will monitor changes on the included CSS (or SCSS) file, and if there is a change on the CSS file, it will recompile the whole CSS (or SCSS) file. As Tailwind CSS 's components and utilities have large file sizes, it will make the recompilation reaaaally slow every time the application.scss file is modified.



 (I am not an expert on this, I might be wrong, please let me know on this)



Here's the diagram explaining the recompilation process :

![recompilation](https://rubyyagi.s3.amazonaws.com/8-solve-slow-compile-tailwind/recompilation.png)



When we change the .red-card part of the application.scss , it cause the whole file to be recompiled again, which includes the heavy tailwind stuff, making it time consuming.



## Workaround

The workaround I used is to **separate it to two CSS (SCSS) files**, one containing the Tailwind CSS stuff, and another one containing my custom @apply CSS, eg: application.scss for the Tailwind CSS stuff, and custom.css for custom @apply CSS.



And then require both of them in the **app/javascript/packs/application.js** file :

```javascript
// app/javascript/packs/application.js

require("@rails/ujs").start()
require("turbolinks").start()

require("stylesheets/application.scss")
require("stylesheets/custom.css")
```

<br>

The **application.scss** file : 

```css
/* application.scss */

@import "tailwindcss/base";
@import "tailwindcss/components";
@import "tailwindcss/utilities";
```

<br>

The **custom.css** file :

```css
/* custom.css */

@layer components {
  .red-card {
    @apply bg-red-400 my-4 w-3/4 mx-auto p-4;
  }

  .red-card p {
    @apply text-white text-xl;
  }
}
```



I have also added the **@layer components** block, so that these custom CSS will be included in the same place as tailwind/components, which is recommended by [Tailwind CSS official documentatio](https://tailwindcss.com/docs/extracting-components#extracting-css-components-with-apply)n : 

> Tailwind will automatically move those styles to the same place as `@tailwind components`, so you don't have to worry about getting the order right in your source files.





With this workaround, I have managed to cut down the recompilation time to **176ms**! This is a lot faster!

![fast recompilation](https://rubyyagi.s3.amazonaws.com/8-solve-slow-compile-tailwind/fast.png)

Happy designing UI with Tailwind CSS!



Also thanks to **Luis Felipe Sanchez** for pointing me to the solution.

 <script async data-uid="4776ba93ea" src="https://rubyyagi.ck.page/4776ba93ea/index.js"></script>