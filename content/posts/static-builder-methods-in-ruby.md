---
title: "Static Builder Methods in Ruby"
date: 2021-07-18T14:56:55-04:00
---
I recently had a discussion about using static methods in ruby as convenience methods to implement the [builder pattern](https://en.wikipedia.org/wiki/Builder_pattern).

I have a strong personal preference for favoring [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) with defaults via the `#initialize` method when working with collaborator objects.
This frequently gives me class definitions that look like this.
<!--more-->

```ruby
class EmailMessage
  def message(content)
    # ...
  end
end

class Messenger
  def initialize(transport: EmailMessage.new)
    self.transport = transport
  end
  
  def message(message_content)
    formatted_message = format_message message_content
    transport.message formatted_message
  end
  
  private
  
  attr_accessor :transport
  
  def format_message(message_content)
    # ...
  end
end
```

Which we then use with the default parameters like this.

```ruby
messenger = Messenger.new
messenger.message "Hello, World!"
```

This gives us the ability to easily swap out message transport implementations with something else.

```ruby
class SlackMessage
  def message(content)
    # ...
  end
end

messenger = Messenger.new transport: SlackMessage.new
messenger.message "Hello, Slack!"
```

We can also use this with test implementations for unit testing the `Messenger` class.

```ruby
require "minitest/mock"

mock = Minitest::Mock.new
mock.expect :message, nil, ["Hello, Test!"]

messenger = Messenger.new transport: mock
messenger.message "Hello, Test!"
mock.verify
```

This leaves us with a very common two line repeating pattern where we instantiate our object, call a method, and then never use the object again.

```ruby
messenger = Messenger.new transport: SlackMessage.new
messenger.message "Hello, Slack!"
```

To simplify this I will commonly add static builder methods on the class which does the object instantiation, calls the method, and discards the return value.

```ruby
class Messenger
  def self.message(message_content, **args)
    new(**args).message(message_content)
  end
  
  # ...
end
```

Which reduces our common usage to a nice one liner.

```ruby
# Using default transport
Messenger.message "Hello, World!"

# Using alternative transport
Messenger.message "Hello, Slack!", transport: SlackMessage.new
```

This feels a lot cleaner to me, however there are a few considerations to keep in mind.

First, this tightly couples the implementation of the `::message` method with the `#message` method.
I am generally comfortable with this coupling as it exists within a single class definition and changes to the signature of the `#message` method can be easily reflected in the `::message` method.

I will also intentionally name the static methods and it's arguments exactly the same as the instance method to make following the code easier.
This pattern does introduce a new layer of indirection and I want to make that as easy to follow as possible.
