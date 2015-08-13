---
layout:     post
title:      Making your own gem server
date:       2015-11-09
summary:    Host your gems locally in a co-working space
---

Ruby Gem bundler process is a very slow process. It takes considerable amount of time to fetch the index from rubygems.org. This slowness interrupts concentration and productivity.

So the idea of having a **local gem server** had been going in my company, Case Commons, for sometime. We use more than 100 open source gems from ruby gems. **Instead of fetching the same data over and over again by every developer, we can cache all the gems to a local server.**

<h4>Advantages</h4>
* Fetching gems from this server will be blazingly fast.
* It will also eliminate our dependency on Rubygems during deploys and other crucial periods. Rubygems has sometime had issues like slow connectivity or no connection.

<h4>Features of the server</h4>
* Storing the local repository of the gems that need to be served.
* Forwarding requests for gems to rubygems.org for the gems it doesn't have.

Let us use <strong>nginx</strong> to serve the gems.

<h4>Steps</h4>
* The gem repository can be called as <strong>local_gems</strong>. All the raw gem files *.gem should be stored in a sub directory <strong>local_gems/gems</strong> directory.

* While being in the ‘local_gems’ repository and outside the ‘gem’ subfolder, make an index of all the available gems on the server.
<pre>
  gem generate_index
</pre>.

* Install nginx on our machine by executing,
<pre>
brew install nginx
</pre>

* Configure nginx to serve our ruby gem repository folder. The nginx configuration is usually stored at <strong>/usr/local/etc/nginx/nginx.conf</strong>. In this configuration file, we want to have something like this,

<pre>
http {
  server {
    location {
      try_files $uri @rubygems;
    }   
    
    location @rubygems {
      proxy_pass https://rubygems.org;
    }
  }
}
</pre>

* Start/reload nginx by
<pre>
sudo nginx / sudo nginx -s reload
</pre>

* Next, point the **Gemfile** to this server. Assuming the local gem server is on port 8080 (the nginx default),
<pre>
  gem sources --remove https://rubygems.org/
  gem sources --add http://localhost:8080
</pre>

Here we go, let us bundle.
<pre>
  bundle
</pre>

And we see that our bundler is **now fetching results from our local server**.

Also we notice, **the gems that are not available on the local server are fetched from the ruby gem server**.

Sample bundler output:

<pre>
Fetching gem metadata from http://localhost:8080/.......
Installing awesome_print 1.2.0
….
….
Your bundle is complete!
Use `bundle show [gemname]` to see where a bundled gem is installed.
</pre>
