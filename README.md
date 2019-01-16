# 🌲 Timber - Great Ruby Logging Made Easy

[![ISC License](https://img.shields.io/badge/license-ISC-ff69b4.svg)](LICENSE.md)
[![Yard Docs](http://img.shields.io/badge/yard-docs-blue.svg)](http://www.rubydoc.info/github/timberio/timber-ruby)
[![Build Status](https://travis-ci.org/timberio/timber-ruby.svg?branch=master)](https://travis-ci.org/timberio/timber-ruby)
[![Code Climate](https://codeclimate.com/github/timberio/timber-ruby/badges/gpa.svg)](https://codeclimate.com/github/timberio/timber-ruby)

Timber for Ruby is a drop in replacement for your Ruby logger that
[unobtrusively augments](https://timber.io/docs/concepts/structuring-through-augmentation) your
logs with [rich metadata and context](https://timber.io/docs/concepts/metadata-context-and-events)
making them [easier to search, use, and read](#get-things-done-with-your-logs). It pairs with the
[Timber console](#the-timber-console) to deliver a tailored Ruby logging experience designed to make
you more productive.

1. [**Installation**](#installation)
2. [**Usage** - Simple & powerful API](#usage)
3. [**Configuration** - Simple & powerful API](#configuration)

## Installation

1. In your `Gemfile`, add the `timber-rails` gem:

    ```ruby
    gem 'timber-rails', '~> 1.0'
    ```

2. Add the Timber initializer in `config/initializers/timber.rb`:

    ```ruby
    config = Timber::Config.instance

    config.integrations.action_view.silence = Rails.env.production?
    ```

3. Add Timber environment configuration:

    ```ruby
    # config/environments/development.rb
    # ...

    # Install the Timber.io logger, send logs over STDOUT. Actual log delivery
    # to the Timber service is handled external of this application.
    logger = Timber::Logger.new(STDOUT)
    logger.level = config.log_level
    config.logger = #{config_set_logger_code}
    ```

    ```ruby
    # config/environments/test.rb
    # ...

    # `nil` is passed to disable logging. It's important to keep the `Timber::Logger`
    # because it provides an API for logging structured data and capturing context.
    logger = Timber::Logger.new(nil)
    logger.level = config.log_level
    config.logger = ActiveSupport::TaggedLogging.new(logger)
    ```

    ```ruby
    # config/environments/production.rb
    # ...
    # Install the Timber.io logger, send logs over STDOUT. Actual log delivery
    # to the Timber service is handled external of this application.
    logger = Timber::Logger.new(STDOUT)
    logger.level = config.log_level
    config.logger = ActiveSupport::TaggedLogging.new(logger)
    ```

## Usage

Use the `Timber::Logger` just like you would `::Logger`:

```ruby
logger.debug("Debug message")
logger.info("Info message")
logger.warn("Warn message")
logger.error("Error message")
logger.fatal("Fatal message")
```

## Configuration

Below are a few popular configuration options, for a comprehensive list, see
[Timber::Config](http://www.rubydoc.info/github/timberio/timber-rails/Timber/Config).

Silence noisy logs that aren't of value to you, just like
[lograge](https://github.com/roidrage/lograge):

```ruby
# config/initializers/timber.rb
Timber.config.logrageify!()
```

It turns this:

```
Started GET "/" for 127.0.0.1 at 2012-03-10 14:28:14 +0100
Processing by HomeController#index as HTML
  Rendered text template within layouts/application (0.0ms)
  Rendered layouts/_assets.html.erb (2.0ms)
  Rendered layouts/_top.html.erb (2.6ms)
  Rendered layouts/_about.html.erb (0.3ms)
  Rendered layouts/_google_analytics.html.erb (0.4ms)
Completed 200 OK in 79ms (Views: 78.8ms | ActiveRecord: 0.0ms)
```

Into this:

```
Get "/" sent 200 OK in 79ms
```

### Pro-tip: Keep controller call logs (recommended)

Feel free to deviate and customize which logs you silence. We recommend a slight deviation
from lograge with the following settings:

```ruby
# config/initializers/timber.rb

Timber.config.integrations.action_view.silence = true
Timber.config.integrations.active_record.silence = true
Timber.config.integrations.rack.http_events.collapse_into_single_event = true
```

This does _not_ silence the controller call log event. This is because Timber captures the
parameters passed to the controller, which are generally valuable when debugging.

For a full list of integration settings, see
[Timber::Config::Integrations](http://www.rubydoc.info/github/timberio/timber-ruby/Timber/Config/Integrations)

### Silence Specific Requests

Silencing noisy requests can be helpful for silencing load balance health checks, bot scanning,
or activity that generally is not meaningful to you. The following will silence all
`[GET] /_health` requests:

```ruby
# config/initializers/timber.rb

Timber.config.integrations.rack.http_events.silence_request = lambda do |rack_env, rack_request|
  rack_request.path == "/_health"
end
```

We require a block because it gives you complete control over how you want to silence requests.
The first parameter being the traditional Rack env hash, the second being a
[Rack Request](http://www.rubydoc.info/gems/rack/Rack/Request) object.

### User Context

By default Timber automatically captures user context for most of the popular authentication
libraries (Devise, and Clearance). See
[Timber::Integrations::Rack::UserContext](http://www.rubydoc.info/github/timberio/timber-rack/Timber/Integrations/Rack/UserContext)
for a complete list.

In cases where you Timber doesn't support your strategy, or you want to customize it further,
you can do so like:

```ruby
# config/initializers/timber.rb

Timber.config.integrations.rack.user_context.custom_user_hash = lambda do |rack_env|
  user = rack_env['warden'].user
  if user
    {
      id: user.id, # unique identifier for the user, can be an integer or string,
      name: user.name, # identifiable name for the user,
      email: user.email, # user's email address
    }
  else
    nil
  end
end
```

*All* of the user hash keys are optional, but you must provide at least one.

### Release Context

[Timber::Contexts::Release](http://www.rubydoc.info/github/timberio/timber-ruby/Timber/Contexts/Release)
tracks the current application release and version.

If you're on Heroku, simply enable the
[dyno metadata](https://devcenter.heroku.com/articles/dyno-metadata) feature. If you are not,
set the following environment variables and this context will be added automatically:

1. `RELEASE_COMMIT` - Ex: `2c3a0b24069af49b3de35b8e8c26765c1dba9ff0`
2. `RELEASE_CREATED_AT` - Ex: `2015-04-02T18:00:42Z`
3. `RELEASE_VERSION` - Ex: `v2.3.1`

All variables are optional, but at least one must be present.
