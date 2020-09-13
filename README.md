# solidus_klaviyo

[![CircleCI](https://circleci.com/gh/solidusio-contrib/solidus_klaviyo.svg?style=svg)](https://circleci.com/gh/solidusio-contrib/solidus_klaviyo)

This extension allows you to integrate your [Solidus](https://solidus.io) store with
[Klaviyo](https://klaviyo.com).

## Installation

Add solidus_klaviyo to your Gemfile:

```ruby
gem 'solidus_klaviyo'
```

Bundle your dependencies and run the installation generator:

```console
$ bundle
$ bundle exec rails g solidus_klaviyo:install
```

The generator will create an initializer at `config/initializers/solidus_klaviyo.rb` with the
default configuration. Take a look at the file and customize it to fit your environment.

## Usage

### Subscribing users to lists

If you want to subscribe a user to a Klaviyo list, the extension provides a handy Ruby API to do
that:

```ruby
SolidusKlaviyo.subscribe_now('YOUR_LIST_ID', 'jdoe@example.com', custom_property: 'value') 
```

We recommend using the built-in background job to subscribe users, in order to avoid blocking your
web workers and slowing down the customer:

```ruby
SolidusKlaviyo.subscribe_later('YOUR_LIST_ID', 'jdoe@example.com', custom_property: 'value')
```

#### Subscribing all users upon signup

If you want to subscribe all users when they sign up, you can just set the `default_list`
configuration option:

```ruby
# config/initializers/solidus_klaviyo.rb
SolidusKlaviyo.configure do |config|
  # ...
  config.default_list = 'klaviyoListId'
end
``` 

Now, all users will be subscribed to the configured list automatically when their account is
created.

### Tracking events

The extension will send the following events to Klaviyo:

- `Started Checkout`: when an order transitions from the `cart` state to `address`.
- `Placed Order`: when an order is finalized.
- `Ordered Product`: for each item in a finalized order.
- `Cancelled Order`: when an order is cancelled.
- `Created Account`: when a user is created.
- `Reset Password`: when a user requests a password reset.

For the full payload of these events, look at the source code of the serializers and events.

#### Implementing custom events

If you have custom events you want to track with this gem, you can easily do so by creating a new
event class and implementing the required methods:

```ruby
module MyApp
  module KlaviyoEvents
    class SubscribedToNewsletter < SolidusKlaviyo::Event::Base
      def name
        'SubscribedToNewsletter'
      end

      def email
        user.email
      end

      def customer_properties
        SolidusKlaviyo::Serializer::Customer.serialize(user)
      end

      def properties
        {
          '$event_id' => user.id.to_s,
          '...' => '...',
        }
      end

      def time
        Time.zone.now
      end

      private

      def user
        payload.fetch(:user)
      end
    end 
  end 
end
```

Once you have created the class, the next step is to register your custom event when initializing
the extension:

```ruby
# config/initializers/solidus_klaviyo.rb
SolidusKlaviyo.configure do |config|
  config.events['subscribed_to_newsletter'] = MyApp::KlaviyoEvents::SubscribedToNewsletter
end
```

Your custom event is now properly configured! You can track it by enqueuing the `TrackEventJob`:

```ruby
SolidusKlaviyo.track_later('signed_up', user: user)
```

*NOTE:* You can follow the same exact pattern to override the built-in events.

### Delivering emails through Klaviyo

If you plan to deliver your transactional emails through [Klaviyo flows](https://help.klaviyo.com/hc/en-us/articles/115002774932-Getting-Started-with-Flows),
you may want to disable the built-in emails that are delivered by Solidus and solidus_auth_devise.

In order to do that, you can set the `disable_builtin_emails` option in the extension's initializer:

```ruby
# config/initializers/solidus_klaviyo.rb
SolidusKlaviyo.configure do |config|
  config.disable_builtin_emails = true
end
```

This will disable the following emails:

- Order confirmation
- Order cancellation
- Password reset

You'll have to re-implement the emails with Klaviyo.

### Test mode

You can enable test mode to mock all API calls instead of performing them:

```ruby
# config/initializers/solidus_klaviyo.rb
SolidusKlaviyo.configure do |config|
  config.test_mode = true
end
```

This spares you the need to use VCR and similar.
 
When in test mode, you can also use our custom RSpec matchers to check if a profile has been
subscribed or an event has been tracked:

```ruby
require 'solidus_klaviyo/testing_support/matchers'

RSpec.describe 'My Klaviyo integration' do
  it 'subscribes users' do
    SolidusKlaviyo.subscribe_now 'my_list_id', 'jdoe@example.com', full_name: 'John Doe'

    expect(SolidusKlaviyo).to have_subscribed('jdoe@example.com')
      .to('my_list_id')
      .with(full_name: 'John Doe')
  end

  it 'tracks events' do
    SolidsuKlaviyo.track_now 'custom_event', foo: 'bar'

    expect(SolidusKlaviyo).to have_tracked_event(CustomEvent)
      .with(foo: 'bar')
  end
end
```

## Development

### Testing the extension

First bundle your dependencies, then run `bin/rake`. `bin/rake` will default to building the dummy
app if it does not exist, then it will run specs. The dummy app can be regenerated by using
`bin/rake extension:test_app`.

```console
$ bundle
$ bin/rake
```

To run [Rubocop](https://github.com/bbatsov/rubocop) static code analysis run

```console
$ bundle exec rubocop
```

When testing your application's integration with this extension you may use its factories.
Simply add this require statement to your spec_helper:

```ruby
require 'solidus_klaviyo/factories'
```

### Running the sandbox

To run this extension in a sandboxed Solidus application, you can run `bin/sandbox`. The path for
the sandbox app is `./sandbox` and `bin/rails` will forward any Rails commands to
`sandbox/bin/rails`.

Here's an example:

```console
$ bin/rails server
=> Booting Puma
=> Rails 6.0.2.1 application starting in development
* Listening on tcp://127.0.0.1:3000
Use Ctrl-C to stop
```

### Releasing new versions

Your new extension version can be released using `gem-release` like this:

```console
$ bundle exec gem bump -v VERSION --tag --push --remote upstream && gem release
```

## License

Copyright (c) 2020 Nebulab Srls, released under the New BSD License.
