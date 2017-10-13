---
layout: post
title:  "The two most common vulnerabilities in Rails (with code)"
date:   2017-10-13 16:00:00
author: "Benoit Larroque"
image:  "/assets/sqreen-cover.png"
excerpt: "Ruby on Rails is one of the most popular frameworks used to create web applications. Although it’s quite safe to use, it’s not perfectly secure."
post_id: "rails-vulnerabilities"
---

# The two most common vulnerabilities in Rails (with code)

Ruby on Rails is one of the most popular frameworks used to create web applications.
It’s very easy to start using it; it shines to do any kind of non-trivial website,
and it is convenient to customize to fit unusual requirements
(which always occur in any project…). It also comes bundled with standard,
sane and safe defaults that make it a breeze to deploy nearly everywhere.

Although it’s quite safe to use, it’s not perfectly secure.
It’s unfortunately far too easy for a developer to introduce vulnerabilities in
Ruby on Rails when not paying attention… In addition, like most web development frameworks,
Rails proposes an endless list of modules to extend capabilities.
A security issue in any of them will let the app open to attackers,
exactly as if the code you produce was flawed.

Let’s look at some very common ones through a very simple blogging application
(built from the [Rails tutorial](http://guides.rubyonrails.org/getting_started.html)).

## Playground

The full source code of this testing app is available on
[Github](https://github.com/sqreen/railsblog) for your perusal
(don’t use it to create your own website, it’s dangerously vulnerable!).
If you want to attack the code vulnerabilities; we have deployed it on
[sqreen-vulnerable-railsblog.herokuapp.com](https://sqreen-vulnerable-railsblog.herokuapp.com/).

We also deployed the exact same version of the app, but this time with
[Sqreen](https://www.sqreen.io/?utm_medium=social&utm_source=blog&utm_campaign=The%20two%20most%20common%20vulnerabilities%20in%20Rails)
– the real-time web application security solution – installed.
You can test how Sqreen protects apps from SQL injections, XSS attacks etc. You can find it at: 
[sqreen-protected-railsblog.herokuapp.com](https://sqreen-protected-railsblog.herokuapp.com).

[Don’t hesitate to give it a try](https://www.sqreen.io/?utm_medium=social&utm_source=blog&utm_campaign=The%20two%20most%20common%20vulnerabilities%20in%20Rails) on your app.
The installation only takes 3 commands:

```
$ echo "gem 'sqreen'" >> Gemfile
$ bundle install
$ echo "token: [TOKEN]" >> config/sqreen.yml
```

## SQL injection

Rails is very good to create [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)
based applications (it can even automagically generate most of the code itself).
What people often miss, though, is a way to search the generated list of records.
Let’s do a simple one:

<img class="post-content__embedded-image" src="/assets/sqreen-article-code-01.png">
<div style="clear: both;"></div>

Unfortunately, as one can guess from the section title this will introduce
an SQL injection vulnerability into your application.
The exploitation of the SQL Injection is trivially done, accessing
the following URL will enable anyone to download the full user database…

```
/articles?utf8=✓&search=+%25%27%29+union+select+id%2C+email+as+title%2C+password+as+text%2C+created_at%2C+updated_at+from+users+where+%28email+like+%27%25
```

<img class="post-content__embedded-image" src="/assets/sqreen-article-code-02.png">
<div style="clear: both;"></div>

When this URL is sent to the server, Rails will helpfully load the article index
(which is also our search page) providing it with a search parameter containing:

```
%') union select id, email as title, password as text, created_at, updated_at from users where (email like '%
```

This gets interpolated into our search string "text like '%#{params[:search]}%'",
which Arel/ActiveRecord will use to create the final query that can be found in Rails log:

<img class="post-content__embedded-image" src="/assets/sqreen-article-code-03.png">
<div style="clear: both;"></div>

This a very well known issue and the
[Rails documentation](http://guides.rubyonrails.org/active_record_querying.html#pure-string-conditions)
even has a warning about it. With a slightly heavier syntax, the issue can be prevented
(look for the array & placeholder condition section next to warning in the documentation).

## Reflected XSS

Let’s continue with our search page, though.

A nice thing to have would be to highlight the part of the query that was matched in
the content of the posts. We could do that in our template, for example using the
[Rails helper](http://api.rubyonrails.org/classes/ActionView/Helpers/TextHelper.html#method-i-highlight).
There are some issues with that, though. First, it’s not very smart at highlighting things.
Second, if the article templates were to be cached, we should not change them before caching.
Fortunately, many client-side solutions will highlight just fine.
We chose to use [mark.js](https://markjs.io/) which has a very simple API:

<img class="post-content__embedded-image" src="/assets/sqreen-article-code-04.png">
<div style="clear: both;"></div>

Since Rails already got a version of jQuery bundled in, we’ll use just that.
We’ll need to pass the exact keywords to highlight in, though.
Let’s do this by reusing part of the sentence we display when performing a search.

Here are the relevant code fragments:

<img class="post-content__embedded-image" src="/assets/sqreen-article-code-05.png">
<div style="clear: both;"></div>

This is nice but, again, there is a vulnerability lurking in there.
It’s a tad more obvious than the previous one but no less dangerous.
To enable us to highlight correctly, we needed the exact same string
that we looked for in the database to pass through to mark.js.
So we used the raw helper that disable the rails XSS protection…

Here an attacker can just send a link containing a nefarious payload e.g.:

```
/articles?utf8=✓&search=<script>alert%28"test"%29<%2Fscript>
```

Like in the previous section when this URL is sent to the server the article search page is loaded,
but this time, the search parameter is set to `<script>alert("test")</script>`
which is then marked safe and interpolated by ERB into our HTML template.
This is sent back to the browser that will read and execute the included javascript.
The Chrome and Safari browsers include a feature called XSS Auditor that will detect and
block reflected XSS client side but it has numerous bypass (a simple proxying script will be enough).

Here we should not have disabled the protection. What we should have done is
parse the escaped string back before using it in our javascript call.

These are only some of the better known and quite easy to demonstrate vulnerabilities in Rails applications.
But Rails apps are targeted every day by new types of SQL injections or XSS attacks and that’s how
[Sqreen](https://www.sqreen.io/?utm_medium=social&utm_source=blog&utm_campaign=The%20two%20most%20common%20vulnerabilities%20in%20Rails)
comes in. Sqreen automatically detects those attacks at runtime and blocks them.
As it doesn’t work with a list of known attacks or patterns Sqreen will even block unknown (or zero-day) attacks.
