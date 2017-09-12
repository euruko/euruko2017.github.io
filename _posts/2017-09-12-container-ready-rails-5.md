---
layout: post
title:  "Container-Ready Rails 5"
date:   2017-09-12 18:00:00
author: "Richard Schneeman"
image:  "/assets/heroku-cover.png"
excerpt: "Rails 5 will be the easiest release ever to get running on Heroku."
post_id: "container-ready-rails-5"
---

# Container-Ready Rails 5

<p>Rails 5 will be the easiest release ever to get running on Heroku. You can get it going in just five lines:</p>

<pre><code>$ rails new myapp -d postgresql
$ cd myapp
$ git init . ; git add . ; git commit -m first
$ heroku create
$ git push heroku master
</code></pre>

<p>These five lines (and a view or two) are all you need to get a Rails 5 app working on Heroku — there are no special gems you need to install, or flags you must toggle. Let&#39;s take a peek under the hood, and explore the interfaces baked right into Rails 5 that make it easy to deploy your app on any modern container-based platform.</p>
<h2>
  Production Web Server as the Default
</h2>

<p>Before Rails 5, the default web server that you get when you run <code>$ rails server</code> is <a href="http://ruby-doc.org/stdlib-2.3.0/libdoc/webrick/rdoc/WEBrick.html">WEBrick</a>, which is the only server that ships with the Ruby standard library. For years now Heroku has <a href="https://devcenter.heroku.com/articles/ruby-default-web-server#why-not-webrick">recommended against using WEBrick as a production webserver</a> mostly due to performance concerns, since by default WEBrick cannot handle more than one request at a time. With the addition of ActionCable to Rails 5, the Rails team needed a web server that could handle concurrent requests, so they decided to make <a href="https://devcenter.heroku.com/articles/deploying-rails-applications-with-the-puma-web-server">Puma webserver</a> the new default. Now, when you deploy a Rails 5 app without a <code>Procfile</code> in your project and Heroku boots your application using <code>$ rails server</code>, you&#39;ll get a performant, production-ready web server <em>by default</em>.</p>

<blockquote>
<p><strong>Note</strong>: if you&#39;re upgrading an existing Rails app, you&#39;ll want to <a href="https://devcenter.heroku.com/articles/deploying-rails-applications-with-the-puma-web-server">manually add Puma to your app</a>.</p>
</blockquote>

<p>In addition to shipping with Puma, Rails also generates <a href="https://github.com/rails/rails/pull/23057">config/puma.rb</a> and efforts were made to <a href="https://github.com/puma/puma/pull/856">allow Puma to read this config file</a> when it&#39;s booted by the <code>$ rails server</code> command. This feature is baked into Puma 3.x+, which allows Rails to configure Puma around the number of threads being generated.</p>

<p>Active Record will generate a pool of five connections by default. These connections are checked out from the pool for the entire duration of the request, so it&#39;s critical that for each concurrent request your webserver can handle, you need that many connections in your connection pool. By default, the Puma server starts with up to 16 threads. This means that it can be processing up to 16 different requests at the same time, but since Active Record is limited to five connections, only five of those requests will have access to the database at a time. This means eventually you&#39;ll hit this error:</p>

<pre><code>ActiveRecord::ConnectionTimeoutError - could not obtain a database connection within 5 seconds. The max pool size is currently 5; consider increasing it
</code></pre>

<p>The solution was to tell Puma that we only want five threads by default. We also wanted a way to re-configure that count without having to commit a change to git, and redeploy for it to take effect. So by default Rails specifies the same number of threads in Puma as Active Record has in its connection pool:</p>

<pre><code class="language-ruby"># config/puma.rb

# Puma can serve each request in a thread from an internal thread pool.
# The `threads` method takes a minimum and maximum.
# Any libraries that use thread pools should be configured to match
# the maximum value specified for Puma. Default is set to 5 threads for minimum
# and maximum, this matches the default thread size of Active Record.

threads_count = ENV.fetch(&quot;RAILS_MAX_THREADS&quot;) { 5 }.to_i
threads threads_count, threads_count
</code></pre>

<p>Note: For a production service there is <a href="https://github.com/rails/rails/pull/24227#issuecomment-198000472">little benefit to setting a minimum thread value</a>.</p>

<p>Now when you deploy, your Puma thread count will match your Active Record thread count so you won&#39;t get timeout errors. Later <a href="https://github.com/rails/rails/pull/23528">the default for Active Record was adjusted</a> to take advantage of the <code>RAILS_MAX_THREADS</code> environment variable. When you scale your Puma thread count via that environment variable, the Active Record connection pool automatically does the right thing.</p>
<h2>
  Port
</h2>

<p>On Heroku, we recommend you specify how to run your app via the <a href="https://devcenter.heroku.com/articles/procfile">Procfile</a> — if you don&#39;t specify a Procfile we will set a default process type for you. Since Heroku apps run inside containers, they need to know which port to connect to, so we set the <code>$PORT</code> environment variable. The buildpack will specify a web process command if you don&#39;t provide one. For example, if you&#39;re deploying a Rails 2 app without a <code>Procfile</code>, by default your app would run:</p>

<pre><code>$ bundle exec ruby script/server -p $PORT
</code></pre>

<p>In Rails 5 you can now use the <code>$PORT</code> <a href="https://github.com/rails/rails/pull/21267">environment variable to specify what port you want your app to connect to</a>. This change doesn&#39;t really affect how your app runs on Heroku, but if you&#39;re trying to run inside of a logic-less build system it can help make it easier to get your application to connect to the right place.</p>
<h2>
  Serving Files by Default
</h2>

<p>Prior to Rails 4.2, a Rails app would not serve its own assets. It was assumed that you would always deploy behind some other kind of server such as NGINX that would serve your static files for you. This is still the default behavior, however, new apps can have the <a href="https://github.com/rails/rails/pull/18100">static file service turned on via an environment variable</a>.</p>

<pre><code class="language-ruby"># config/environments/production.rb

config.public_file_server.enabled = ENV[&#39;RAILS_SERVE_STATIC_FILES&#39;].present?
</code></pre>

<p>Heroku will set this value when you deploy a Ruby app via the <a href="https://github.com/heroku/heroku-buildpack-ruby">Heroku Ruby Buildpack</a> for Rails 4.2+ apps. Previously you would have to either set this value manually or use the <a href="https://github.com/heroku/rails_12factor">rails_12_factor</a> gem.</p>
<h2>
  STDOUT Logging
</h2>

<p>The default logging location in Rails has always been to a file with the name of your environment so production logs go to <code>logs/production.log</code>. This works well for a traditional deployment but when deploying to a container-based architecture, it makes retrieving and aggregating logs very difficult. Instead, Heroku has advocated for <a href="http://12factor.net/logs">logging to STDOUT instead</a> and treating your logs as streams. These streams can then be directly consumed, fed into a <a href="https://elements.heroku.com/addons#logging">logging add-on</a> for archival, or even used for structured data aggregation.</p>

<p>The default hasn&#39;t changed, but starting in Rails 5, new apps can log to STDOUT via an environment variable</p>

<pre><code class="language-ruby">if ENV[&quot;RAILS_LOG_TO_STDOUT&quot;].present?
  logger           = ActiveSupport::Logger.new(STDOUT)
  logger.formatter = config.log_formatter
  config.logger = ActiveSupport::TaggedLogging.new(logger)
end
</code></pre>

<p>This value can be set by the container or the platform on which your Rails app runs. In our case, the Ruby buildpack detects your Rails version, and if it&#39;s Rails 5 or greater will set the <code>RAILS_LOG_TO_STDOUT</code> environment variable.</p>
<h2>
  DATABASE_URL
</h2>

<p>Support for connection to the database specified in <code>$DATABASE_URL</code> has been around since Rails 3.2, however, there were a large number of <a href="https://github.com/rails/rails/pull/13578">bugs and edge cases that weren&#39;t completely handled until Rails 4.1</a>. Prior to Rails 4.1, because the DATABASE_URL integration was not 100% of the way there, Heroku used to write over your <code>config/database.yml</code> with a file that parsed the environment variable and returned it back as in YAML format. You can see the <a href="https://github.com/heroku/heroku-buildpack-ruby/blob/a309797a663ca2cf103591aa31caa4bf6dc92e59/lib/language_pack/ruby.rb#L680-L734">contents of the &quot;magic&quot; database.yml file here</a>. The biggest problem is that this magic file replacement wasn&#39;t expected. People would add config keys for things like <code>pool</code> which specifies your Active Record connection pool, and it would be silently ignored. So they had to resort to hacks like this code to modify the database configuration</p>

<pre><code class="language-ruby"># Hack, do not use with Rails 4.1+

Rails.application.config.after_initialize do
  ActiveRecord::Base.connection_pool.disconnect!

  ActiveSupport.on_load(:active_record) do
    config = ActiveRecord::Base.configurations[Rails.env] ||
                Rails.application.config.database_configuration[Rails.env]
    config[&#39;pool&#39;]              = ENV[&#39;DB_POOL&#39;]      || ENV[&#39;MAX_THREADS&#39;] || 5
    ActiveRecord::Base.establish_connection(config)
  end
end
</code></pre>

<p>Even then, you need to make sure that code gets run correctly in all different ways your app can be booted. For example, if you&#39;re preloading your app to take advantage of Copy on Write, you&#39;ll need to make sure this code runs in an &quot;after fork&quot; block. While it works around the issue, it normally meant that configuration was spread around an application in many places, and often resulted in different behaviors for different types of dynos.</p>

<p>After the 4.1 patch, Rails merged configuration from the <code>config/database.yml</code> and the <code>$DATABASE_URL</code> environment variable. Heroku no longer needed to over-write your checked-in file, so you can now set pool size directly in your <code>database.yml</code> file. You can see the <a href="https://devcenter.heroku.com/articles/rails-database-connection-behavior#configuring-connections-in-rails-4-1">database connection behavior in Rails 4.1 and beyond explained here</a>.</p>

<p>This allows anyone who does not need to configure a database via an environment variable to run exactly as before, but now anyone connecting using the environment variable can keep additional Active Record config in one canonical location. </p>
<h2>
  SECRET_KEY_BASE
</h2>

<p>At around the time that Rails 4.1 introduced <code>$DATABASE_URL</code> support, Rails was introducing the secret token store as a new feature. Prior to this feature, there was one secure string that was used to prevent <a href="https://en.wikipedia.org/wiki/Cross-site_request_forgery">Cross-site request forgery (CSRF)</a>. Lots of developers forgot that it was in their source, and they would check that into their git repository. It&#39;s never a good idea to store secrets in source control, and quite a few applications that were public on GitHub were vulnerable as a result. Now with the introduction of the secret key store, we can <a href="https://github.com/rails/rails/pull/13703">set this secret token value</a> with an environment variable.</p>

<pre><code class="language-yaml"># Do not keep production secrets in the repository,
# instead read values from the environment.
production:
  secret_key_base: &lt;%= ENV[&quot;SECRET_KEY_BASE&quot;] %&gt;
</code></pre>

<p>Now we do not need to check secure things directly into our application code. With new Rails 4.1+ apps you are required to provide a secret via the <code>SECRET_KEY_BASE</code> environment variable, or to set the value some other way.</p>

<p>When deploying a Rails 4.1+ app, Heroku will specify a <code>SECRET_KEY_BASE</code> on your app by default. It is a good idea to rotate this value periodically. You can see the current value by running</p>

<pre><code>$ heroku run bash
Running bash on issuetriage... up, run.8903
~ $ echo $SECRET_KEY_BASE
abcd12345thisIsAMadeUpSecretKeyBaseforThisArticle
</code></pre>

<p>To set a new key you can use</p>

<pre><code>$ heroku config:set SECRET_KEY_BASE=&lt;yournewconfigkeyhere&gt;
</code></pre>

<p>Note: That this may mean that people who are submitting a form in the time between the key change will have an invalid request as the CSRF token will have changed.</p>
<h2>
  Safer Database Actions
</h2>

<p>One of the scariest things you can say to a co-worker is &quot;I dropped the production database&quot;. While it doesn&#39;t happen often, it&#39;s a serious enough case to warrant an extra layer of protection. In Rails 5, the database is now aware of the environment that it is run in and by default <a href="https://github.com/rails/rails/pull/22967">destructive actions will be prevented on production database</a>. This means if you are connected to your &quot;production&quot; database and try to run</p>

<pre><code>$ rake db:drop
</code></pre>

<p>Or other destructive actions that might delete data from your database you&#39;ll get an error.</p>

<pre><code>You are attempting to run a destructive action against your &#39;production&#39; database
if you are sure you want to continue, run the same command with the environment variable
DISABLE_DATABASE_ENVIRONMENT_CHECK=1
</code></pre>

<p>While not required to run on Heroku, it&#39;s new in Rails 5, and might save you from a minor catastrophe one day. If you&#39;re <a href="https://devcenter.heroku.com/articles/heroku-postgres-plans#plan-tiers">running on a high enough Postgres plan tier</a>, you&#39;ll also have the ability to <a href="https://devcenter.heroku.com/articles/heroku-postgres-rollback">rollback a database to a specific point in time if anything goes wrong</a>. This is currently available for different durations for all plans Standard and above.</p>
<h2>
  Request IDs
</h2>

<p>Running a Rails app with high traffic can be demanding, especially when you can&#39;t even tell which of your log lines go together with a single Request. For example three requests could look something like this in your logs:</p>

<pre><code>Started GET &quot;/&quot; for 72.48.77.213 at 2016-01-06 20:30:21 +0000
  Rendered welcome/index.html.erb within layouts/application (0.1ms)
Started GET &quot;/&quot; for 72.48.77.213 at 2016-01-06 20:30:22 +0000
Started GET &quot;/&quot; for 72.48.77.213 at 2016-01-06 20:30:23 +0000
  Rendered welcome/index.html.erb within layouts/application (0.1ms)
Processing by WelcomeController#index as HTML
Completed 200 OK in 5ms (Views: 3.8ms | ActiveRecord: 0.0ms)
Processing by WelcomeController#index as HTML
  Rendered welcome/index.html.erb within layouts/application (0.1ms)
Completed 200 OK in 5ms (Views: 3.8ms | ActiveRecord: 0.0ms)
  Processing by WelcomeController#index as HTML
Completed 200 OK in 5ms (Views: 3.8ms | ActiveRecord: 0.0ms)
</code></pre>

<p>With Rails 5, the request ID will be logged by default, ensuring each request is tagged with a unique identifier. While they are still interleaved it is possible to figure out which lines belong to which requests. Like:</p>

<pre><code>[c6034478-4026-4ded-9e3c-088c76d056f1] Started GET &quot;/&quot; for 72.48.77.213 at 2016-01-06 20:30:21 +0000
[c6034478-4026-4ded-9e3c-088c76d056f1]  Rendered welcome/index.html.erb within layouts/application (0.1ms)
[abuqw781-5026-6ded-7e2v-788c7md0L6fQ] Started GET &quot;/&quot; for 72.48.77.213 at 2016-01-06 20:30:22 +0000
[acfab2a7-f1b7-4e15-8bf6-cdaa008d102c] Started GET &quot;/&quot; for 72.48.77.213 at 2016-01-06 20:30:23 +0000
[abuqw781-5026-6ded-7e2v-788c7md0L6fQ]  Rendered welcome/index.html.erb within layouts/application (0.1ms)
[c6034478-4026-4ded-9e3c-088c76d056f1] Processing by WelcomeController#index as HTML
[c6034478-4026-4ded-9e3c-088c76d056f1] Completed 200 OK in 5ms (Views: 3.8ms | ActiveRecord: 0.0ms)
[abuqw781-5026-6ded-7e2v-788c7md0L6fQ] Processing by WelcomeController#index as HTML
[abuqw781-5026-6ded-7e2v-788c7md0L6fQ]  Rendered welcome/index.html.erb within layouts/application (0.1ms)
[abuqw781-5026-6ded-7e2v-788c7md0L6fQ] Completed 200 OK in 5ms (Views: 3.8ms | ActiveRecord: 0.0ms)
[acfab2a7-f1b7-4e15-8bf6-cdaa008d102c]  Processing by WelcomeController#index as HTML
[acfab2a7-f1b7-4e15-8bf6-cdaa008d102c] Completed 200 OK in 5ms (Views: 3.8ms | ActiveRecord: 0.0ms)
</code></pre>

<p>Now, if you have the logs and you find this unique ID, you can filter to only look at information from that request. So a filtered log output would be very clear:</p>

<pre><code>[c6034478-4026-4ded-9e3c-088c76d056f1] Started GET &quot;/&quot; for 72.48.77.213 at 2016-01-06 20:30:21 +0000
[c6034478-4026-4ded-9e3c-088c76d056f1]  Rendered welcome/index.html.erb within layouts/application (0.1ms)
[c6034478-4026-4ded-9e3c-088c76d056f1] Processing by WelcomeController#index as HTML
[c6034478-4026-4ded-9e3c-088c76d056f1] Completed 200 OK in 5ms (Views: 3.8ms | ActiveRecord: 0.0ms)
</code></pre>

<p>In addition to this benefit, the request can be set via the <code>X-Request-ID</code> header so that the same request could be traced between multiple components. For example, a request comes in from the Heroku router which assigns a <a href="https://devcenter.heroku.com/articles/http-request-id">request id</a>. As the request is processed we can log that id, then when the request is passed on to Rails, the same id is used. That way if a problem is determined to be not caused in Rails, it could be traced back to other components with the same ID. This default was added in <a href="https://github.com/rails/rails/pull/22949">PR #22949</a>.</p>

<p>This is another feature that isn&#39;t explicitly required to run on Heroku, however, it will make running an application at scale much easier.</p>
<h2>
  Summary
</h2>

<p>Rails 5 is the easiest to use Rails version on Heroku ever. We also hope that it&#39;s the easiest version to run anywhere else. We&#39;re happy that the power of &quot;convention over configuration&quot; can be leveraged by container-based deployment platforms to provide a seamless production experience. Many of these features listed such as request IDs and destructive database safeguards are progressive enhancements that will help all app developers regardless of where they deploy or how they run in production. Heroku has been committed to providing the best possible Ruby and Rails experience from its inception, whether that means building out platform features developers need, automating tasks via the buildpack, or working with upstream maintainers. While we want to provide an easy experience, we don&#39;t want one that is <a href="https://blog.codeship.com/programming-magic/">too &quot;magical&quot;</a>. By working together in open source we can make software easier to deploy and manage for all developers, not just Heroku customers.</p>

<p>If you haven&#39;t already, try <a href="https://blog.heroku.com/archives/2016/1/22/rails-5-beta-upgrade">upgrading to Rails 5 beta</a>.</p>

<p>Check out this Dev Center article for more information on <a href="https://devcenter.heroku.com/articles/getting-started-with-rails5">getting started with Rails 5.x on Heroku</a>.</p>


<p>
  This blog post was originally published on
  <a href="https://blog.heroku.com/container_ready_rails_5">blog.heroku.com</a>.
</p>

