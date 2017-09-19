---
layout: post
title:  "Two Welcome Side Effects of Webpack in Rails"
date:   2017-09-19 10:00:00
author: "Amin Shah Gilani"
image:  "/assets/toptal-cover.png"
excerpt: "I began working with the back-end with Node, back in the Node v0.10.x days when the React vs. Angular debate was very new and very, very hot. I was getting very tired of copying boilerplate code when someone introduced me to Ruby on Rails 4, and the idea of convention over configuration. A few months later, I was hooked. It was suddenly like being able breathe freely. Writing back-end code becomes quick and trivial."
post_id: "two-welcome-side-effects-of-webpack-in-rails"
---

# Two Welcome Side Effects of Webpack in Rails

I began working with the back-end with Node, back in the Node v0.10.x days when the React vs. Angular debate was very new and very, very hot. I was getting very tired of copying boilerplate code when someone introduced me to Ruby on Rails 4, and the idea of convention over configuration. A few months later, I was hooked. It was suddenly like being able breathe freely. Writing back-end code becomes quick and trivial.

Ruby on Rails is great for prototyping and building an MVP, in fact, being a freelance software engineer at [Toptal](http://toptal.com/top-3-percent?utm_source=euroko2017HU&utm_medium=logo&utm_campaign=sponsorships), I get to work with clients of all sizes, building web applications to solve interesting problems. However, when you start doing anything advanced on the front-end, you realize that, here, Rails conventions become a bit basic. The good news is that Rails 5.1 fixes everything with Yarn and Webpack.

Yarn and Webpack. Ruby on Rails 5.1 ships with the two things you need to make a wonderful front-end, with your favorite front-end framework, and it gets rid of the jQuery dependency. It's official, Ruby on Rails is now a wonderful framework, not just on the back-end, but the front-end as well!

The [Rails/Webpacker](https://github.com/rails/webpacker) description would have you believe that this tool is meant to be used only for front-end application frameworks:

> Webpacker makes it easy to use the JavaScript pre-processor and bundler [Webpack 3.x.x+](https://webpack.js.org/) to manage application-like JavaScript in Rails. It coexists with the asset pipeline, as the primary purpose for Webpack is app-like JavaScript, not images, CSS, or even JavaScript Sprinkles (that all continues to live in app/assets).

But I'm here to tell you otherwise. A side-effect of Webpack is that it fixes my two pet peeves with Ruby on Rails:

1. The hacks to support front-end dependencies
2. The inability to use URL helpers in third-party CSS dependencies

## Managing Front-end Dependencies with Yarn

The current asset pipeline requires a bit of hacking to support even the dependency management behavior, and pure CSS libraries that link assets simply aren't compatible with Rails without rewriting declaring their paths with [helpers](http://guides.rubyonrails.org/asset_pipeline.html#css-and-sass) because of the way Rails renames and adds digests to asset paths in production. Historically, the way to get around this has been to use a CDN. More on this later.

In the past, if you really needed Bootstrap, you had four ways to include it in your project:

- [Rails-Assets.org](https://rails-assets.org/): this third-party gem server is a proxy to install Bower dependencies via bundler. Add the following like to your `Gemfile`:

```ruby
gem 'rails-assets-bootstrap', source: 'https://rails-assets.org'
```

Require the minified version to your `application.js`:

```javascript
//= require bootstrap
```

And then require the minified version to your `application.css`

```
*= require bootstrap
```

- Use a CDN: the easiest way to get a dependency was to load your dependency from a CDN by adding the following to your view or layout:

```html
<!-- Latest compiled and minified CSS -->
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">

<!-- Optional theme -->
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap-theme.min.css" integrity="sha384-rHyoN1iRsVXV4nD0JutlnGaslCJuC7uwjduW9SVrLvRYooPp2bWYgmgJQIXwl/Sp" crossorigin="anonymous">

<!-- Latest compiled and minified JavaScript -->
<script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/js/bootstrap.min.js" integrity="sha384-Tc5IQib027qvyjSMfHjOMaLkfuWVxZxUPnCJA7l2mCWNIpG9mGCD8wGNIcPD7Txa" crossorigin="anonymous"></script>
```

- Install a package manager:

If you were serious about dependency management with the right tools, you could either install and initialize Bower or NPM and set them up yourself, use the [`bower-rails`](https://github.com/rharriso/bower-rails) [`npm-rails`](https://github.com/endenwer/npm-rails) gem, initialize an NPM package, and

- Copy dependency code to `vendors`: The _worst_ and most common thing to do was to copy and paste the third-party code to the `vendors` directory and include that in your asset pipeline.

These took time to implement, and felt incredibly hacky, **not anymore!**

With Rails 5.1, Yarn support is baked right into the framework! The power of [NPM repository](https://www.npmjs.com/) is now at your fingertips. Oh, you want Bootstrap on your framework?

```bash
$ yarn add bootstrap
yarn add v0.27.5
warning ../../package.json: No license field
info No lockfile found.
[1/4] Resolving packages...
[2/4] Fetching packages...
[3/4] Linking dependencies...
[4/4] Building fresh packages...
success Saved lockfile.
success Saved 1 new dependency.
└─ bootstrap@3.3.7
Done in 1.72s.
```

This command adds the package to your `package.json` file and installs to the local `node_modules` folder.

```
--- a/package.json
+++ b/package.json
@@ -1,5 +1,7 @@
 {
   "name": "whazzap",
   "private": true,
-  "dependencies": {}
+  "dependencies": {
+	"bootstrap": "^3.3.7"
+  }
 }
```

To include Bootstrap in your app, require the path of your files relative to `node_modules`. In this case, since the compiled Bootstrap files are at `node_modules/bootstrap/dist/`, we make the following changes:

```
# app/assets/javascripts/application.js
+//= require bootstrap/dist/js/bootstrap.min.js

# app/assets/stylesheets/application.css
+ *= require bootstrap/dist/css/bootstrap.min.css
```

At this point, Bootstrap should be fully loaded and working for your app. This is wonderful!

## Pure CSS Libraries without Rails Helpers

Rails just doesn't play well with CSS libraries. If a library links to contained assets using a regular CSS `url()` declaration, the asset won't load in production---i.e, if your library sets the background image in CSS using `background-image: url('background.png');`  it won't work in production because the image's public path will become `background-e96ed6bdb4a9942747db0179ec8aeb9d995fdf06cb5c2caeaea5ceb7651ac2fd.png`. The only way to fix this with the Rails asset pipeline is to rewrite the CSS with `background-image: image-url('background.png');`  which you can't do at all since your library is fetched via `yarn`.

Webpack allows you to get around this!

The short version is that you can import CSS in your webpacks. Let's use Semantic UI CSS as an example, which links to Font Awesome using:

```scss
src: url("./themes/default/assets/fonts/icons.eot");
```

And there's no way for you to edit this line.

With Webpack, a way to get around this is to import the CSS not in your `app/assets/stylesheets/application.css`, but instead in `app/javascript/packs/application.js` such as below:

```javascript
import 'semantic-ui-css/semantic';
```

Then require this pack as a stylesheet asset in your view with:

```erb
<%= stylesheet_pack_tag "stylesheets", :media => 'all' %>
```

## Closing thoughts and acknowledgements

Rails has once again become a very modern web framework. This change has me more excited than websockets support in Rails 5 through Action Cable.

I’d like to acknowledge the wonderful community of [Ruby developers](toptal.com/ruby) at Toptal. Without the fun discussions I have with them, I would never have been driven to explore the interesting ways to use Webpack, and these tricks would never have been discovered.
