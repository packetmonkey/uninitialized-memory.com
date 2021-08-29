---
title: "Always Return the Same Thing"
date: "2021-08-29T14:34:58-04:00"
---

It's a cliche that the answer to any given programming question is "it depends" and hard rules that you should 100% follow should always be questioned as there is usually an exception somewhere.
This means when I do have a rule I feel I should follow 100% of the time it's worth special notice.

In the spirit of [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

> All functions or methods you write **MUST** return an identical type regardless of their success or failure.

The Ruby standard library often doesn't follow this principal and it can cause problems.
Looking at the [Hash#[]](https://docs.ruby-lang.org/en/3.0.0/Hash.html#method-i-5B-5D) method for example

```ruby
hash = { answer: 42 }

hash[:answer]    # => 42
hash[:other_key] # => nil
```

If a [default value](https://docs.ruby-lang.org/en/3.0.0/Hash.html#class-Hash-label-Default+Values) for the hash hasn't been set, we return `nil` instead of the value we wanted.
For this specific reason I will always use [Hash#fetch](https://docs.ruby-lang.org/en/3.0.0/Hash.html#method-i-fetch) which will either throw a `KeyError` exception or allows me to return a default value.
This prevents our application from leaking a `nil` into our program which can result in a `NoMethodError` when we try to use that `nil` in a context where one was not expected.

[Avdi Grimm](https://avdi.codes) wrote a [whole book](https://secure.pragprog.com/titles/agcr/confident-ruby/) on dealing with this and similar problems that is worth a look.

This isn't just a Ruby problem, Java libraries will often return `null` as an error which can result in the common `NullPointerException` when you forget to check.

If it's Ruby's `nil`, Java's `null`, or something similar, we are really talking about [Tony Hoare's billion dollar mistake](https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare/).


Many languages have their own way to deal with this idiomatically. 
Go functions will return a tuple containing the expected result and an error object that you must inspect to address this.
Rust's standard library makes heavy use of the `Option` or `Result` types for when there may be no result or an error occurs.
In Ruby I'll frequently use the [dry-monads](https://dry-rb.org/gems/dry-monads/1.3/) gem to provide a similar interface without having to write the boilerplate myself.

Whenever the result of a function or method is uncertain, you need to return a type that models that uncertainty. 
Expecting someone, even yourself, to always remember to check for a `nil` is guaranteed to result in problems _eventually_.

# What about exceptions?
I'm ignoring exceptions for the moment except to say that exceptions for error handling I feel is fundamentally flawed as it introduces an alternative control flow which complicates the application logic.
If you have a truly unrecoverable error exceptions may give you a semantically appropriate way to end the program but if you think that the error could be handled, it should be modeled appropriately allowing the developer calling your function to handle the result in a consistent way. 
