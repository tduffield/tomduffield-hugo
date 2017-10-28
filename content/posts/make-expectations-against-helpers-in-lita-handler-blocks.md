---
title: "Make expectations against helpers in Lita Handler Blocks"
date: 2017-02-21
tags: [ programming ]
---

In the past several weeks I've been doing some personal coding on several [Lita](https://www.lita.io/) plugins for the teams that I work on at Chef. Lita plugins are relatively new to me, but they've been a neat way to get some personal coding in whilst also bringing some value to my team. Some of these plugins are small, but others are large and rather complex, requiring helper functions, classes, or sometimes entire gems. What I want to talk about today is a weird quick that exists in Lita v4.7.1 that can impact how you make assertions against test instances of your Handler class. 

I'm a huge proponent of testing, but one of the major issues that I had faced up until recently in [my Lita Handler development](https://docs.lita.io/plugin-authoring/handlers/) is how I could make assertions about helper functions that are called within Lita Route blocks. For example, let's say that I have the following [Chat Route](https://docs.lita.io/plugin-authoring/handlers/#chat-routes):

```ruby
route(/^complex\s+command\s+(.+)/) do |response|
  message_to_user = some_helper_method(response[0])
  response.reply(message_to_user)
end

def some_helper_method(message)
  # something very complex, or something that I don't want/need handled by the Chat Route
end
```

Now, I want to test `some_helper_method` in isolation from the route itself, so it doesn't make sense for me to write the tests against the Chat Route (there are many reasons that this could happen, so just follow along for now). I want to make an assertion that my helper method `some_helper_method` is being called from the chat route and the return value of that helper is handled in a certain way. My assumption was that because `some_helper_method` was an instance method on the Handler class, it could make a simple expectation:

```ruby
# There is some Lita RSpec helper magic that gives you subject, an instance of the described Handler class.

describe "complex command" do
  it "calls out to helper method" do
    expect(subject).to receive(:some_helper_command).with("foo").and_return("bar")
    send_command("complex command foo")
    expect(replies.last).to eql("bar")
  end
end
```

There was, sadly, only one problem with this: I would constantly get a failure on my expectation that `some_helper_method` would be called. This confused me for a very long time because Pry would reveal that the instance method was indeed there. I ended up avoiding the problem for a long time until I finally came across the answer browsing the source code: [The code inside the block is actually being executed against a new instance of the handler!](https://github.com/litaio/lita/blob/v4.7.1/lib/lita/handler/chat_router.rb#L94-L97) When I realized this the solution became clear: I needed to make an assertion not against the top level instance of the handler, but any instance of the handler.

```ruby
describe "complex command" do
  it "calls out to helper method" do
    expect_any_instance_of(described_class).to receive(:some_helper_command).with("foo").and_return("bar")
    send_command("complex command foo")
    expect(replies.last).to eql("bar")
  end
end
```

This worked like a charm, and I was able to clean up my Lita plugin code quite a bit. 