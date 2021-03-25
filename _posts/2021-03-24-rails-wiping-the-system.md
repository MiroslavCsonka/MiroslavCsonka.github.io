---
title: Rails cache wiping the system
classes: wide
---

## The problem

Let's examine a little snippet

{% include figure image_path="/assets/rails-redis-cache-bug/hero.png" alt="Summary of the bug" caption="Summary of the bug" %}

Do you see how it wipes every single Sidekiq job and its statistics? How your recurring Sidekiq jobs are gone?
How [Flipper](https://github.com/jnunemaker/flipper) feature flags are missing? No? Well then keep on reading.

## How and why the bug got introduced?

Imagine you are running your app on [Heroku](https://www.heroku.com). Run a few commands, you are up and running, and everything is great. As you add features, you start needing background jobs. Not a problem. You add [Redis addon](https://elements.heroku.com/addons/heroku-redis) with a few clicks, configure [Sidekiq](http://sidekiq.org), and you are back to building features.

One day, you realize you could benefit from Rails caching, so you head to [documentation](https://guides.rubyonrails.org/caching_with_rails.html#activesupport-cache-rediscachestore) and you see that a single line of code is all it takes to have cache in Redis:

```ruby
config.cache_store = :redis_cache_store, { url: ENV['REDIS_URL'] }
```

Awesome! This is why you love Ruby. Everything is pre-baked, works together, and you focus on features.

Later on, you start seeing some odd behavior.

## How the bug manifests itself

Your Sidekiq recurring jobs are missing. Sidekiq statistics that usually look like this:
![sidekiq with statistics](/assets/rails-redis-cache-bug/sidekiq-with-statistics.png)
are now blank.
![sidekiq without statistics](/assets/rails-redis-cache-bug/sidekiq-without-statistics.png)

Your system behaves as if all feature flags are turned off.

Your [Sidekiq recurring jobs](https://github.com/Moove-it/sidekiq-scheduler) are missing. But when you re-deploy your app they are right back.

Some of these things happen only on staging, while others on production. Of course, you don't notice all of them at once, but within a few days, you connect the dots. Something is off.

## How we figured out what's going on

I was super lucky that [@pawelpacana](https://twitter.com/pawelpacana) was free to pair on this. Without him, I would be lost for hours. together we found the bug in maybe 20 minutes. Use that against "pairing isn't economical" when dealing with managers 😉

After seeing all of this odd behavior, you piece it together and realize that all of these are backed by Redis. The 2 main culprits could be Sidekiq and Rails cache. We were running Sidekiq for a few months so it couldn't be it. So we started digging into Rails cache.

We only used 2 methods:
* `Rails.cache.fetch` - to read and/or populate single key
* `Rails.cache.clear` - to clear the whole cache


I jumped into the declaration of `Rails.cache.clear` with my handy find declaration feature in Rubymine (`CMD+B`) I find the implementation within seconds
![Rubymine go to declaration](/assets/rails-redis-cache-bug/rubymine-go-to-declaration.png)

And we see this:
```ruby
# Clear the entire cache on all Redis servers. Safe to use on
# shared servers if the cache is namespaced.
#
# Failsafe: Raises errors.
def clear(options = nil)
  failsafe :clear do
    if namespace = merged_options(options)[:namespace]
      delete_matched "*", namespace: namespace
    else
      redis.with { |c| c.flushdb }
    end
  end
end
```
Snippet taken from [rails/activesupport/lib/active_support/cache/redis_cache_store.rb](https://github.com/rails/rails/blob/main/activesupport/lib/active_support/cache/redis_cache_store.rb#L307-L319).

And there you have it, handy Ruby comment that explains exactly what's going on and gives you a hint on how to fix it.

## Solution

There are 2 ways to fix this. Use separate Redis instances for Rails Cache, or configure Redis namespace.

For simplicity, we opted for a Redis namespace. This tells Rails to put its items within a namespace and it'll clear only that namespace when `Rails.cache.clear` is called

```ruby
# config/application.rb
config.cache_store = :redis_cache_store, { url: ENV["REDIS_URL"], namespace: "rails" }
```

{% include newsletter_form.html %}
