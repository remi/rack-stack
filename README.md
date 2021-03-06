RackStack
=========

> **Here Be Dragons** RackStack is currently in ALPHA.  This is a preview.

`RackStack` is a managed stack of Rack applications (*inspired by [Rack::Builder][]*)

Installation
------------

RackStack is intentionally not distrubuted on RubyGems *YET*.

For now, you can use Bundler to use RackStack from git:

```ruby
gem "rack-stack", :git => "https://github.com/remi/rack-stack.git"
```

Usage
-----

```ruby
require "rack/stack" # or "rack-stack"

rack_stack = RackStack.new do
  use MyMiddleware
  map "/admin" do
    run AdminApp.new
  end
  run MyApp.new
end

# A RackStack instance is a Rack application, so you can #call it.
status, headers, body = rack_stack.call(env)
```
If you're familar with Rack::Builder, that should look very familiar!

RackStack is actually intended to be [compatible with Rack::Builder (and Rack::URLMap)][compatibility].

RackStack offers a number of additional features:

 1. [Conditional Logic](#conditional-logic)
 1. [Named Components](#named-components)
 1. [Use as Middleware](#use-as-middleware)
 1. [Stack Manipulation](#stack-manipulation)

Use Case
--------

I dunno.  Are there good use cases?  How would you find RackStack useful?

Personally, I like to use RackStack to make it easy to mount mock web APIs in tests, eg. when using [Artifice][].  ([RackStack Artifice sample](https://github.com/remi/rack-stack/tree/master/spec/sample-use-cases/artifice))

Any scenario where you want to mount many Rack applications together into 1 application is a possible use case for RackStack.

RackStack can also be used as a Rack Router (by making use of `:when` conditions and/or `#map` statements).

Please share your use cases!  Also, if there's a better existing tool that solves these problems, I'd love to know!  Until then, enjoy RackStack!

Conditional Logic
-----------------

RackStack allows you to easily add conditional logic for `:when` to `#run`, `#use`, or `#map` a Rack component.

```ruby
RackStack.new do

  # When a block is given with 1 argument, the current request will be yielded (as a Rack::Request)
  use MyMiddleware, when: ->(request){ request.path_info =~ /about-us/ }

  # When a block is given with no arguments, the block is evaluated against the current request instance
  use MyMiddleware, when: ->{ path_info =~ /about-us/ }

  # When a Hash (or Array of pairs) is given, each value will be compared against the value from the 
  # current request.  In this example, the following would be evaluated: /about-us/ === "<the path info>"
  use MyMiddleware, when: { path_info: /about-us/ }

  # Map also works with :when.
  map "/section", when: ->{ params["mobile"] == "true" } do
    # Nested options work with conditionals as well.
    run AndroidSectionApp.new, when: { user_agent: /Android/i }
    run MobileSectionApp.new
  end

  # Run also works with :when.
  # If RackStack handles a request and NONE of the conditionals match, 
  # a NoMatchingApplicationError will be thrown.
  run @app, when: { host: "domain.com" }

end
```

Named Components
----------------

RackStack allows you to name any of your Rack components.

```ruby
@rack_stack = RackStack.new do
  use :cool_middleware, CoolMiddleware
  map :foo, "/foo" do
    run :foo_app, FooApp.new
  end
  run :main, MainApp.new
end
```

### Getting components by name

By providing names for our middleware/maps/endoints, you can easily access 
these instances via `RackStack#get`.

```ruby
# For middleware, the instance of the middleware that we create & use to process requests is returned.
@rack_stack.get(:cool_middleware)
# => #<CoolMiddleware:0x000000015fb520>

# For endpoints, the application instance is returned.
@rack_stack.get(:main)
# => #<MainApp:0x000000015db7a8>

# Components nested within maps are returned too.
@rack_stack.get(:foo_app)
# => #<FooApp:0x000000015fb740>

# For maps, the RackStack instance (representing the nested map) is returned.
@rack_stack.get(:foo)
# => #<RackStack:0x000000015cf840>
```

We also provide some useful shortcuts for `RackStack#get`

```ruby
@rack_stack.get(:cool_middleware)
# => #<CoolMiddleware:0x000000015fb520>

# Get is aliased to [], so this also works.
@rack_stack[:cool_middleware]

# We also have a method_missing implementation, so this also works.
@rack_stack.cool_middleware
@rack_stack.respond_to? :cool_middleware # => true
```

### Removing components

Names may be used to remove components from a RackStack.

```ruby
# RackStack#remove removes every Rack component with the given name from the stack.
@rack_stack.remove :cool_middleware
```

Use as Middleware
-----------------

A Rack application generated by a Rack::Builder can only be run as a Rack endpoint,
not as a middleware.

RackStack can be run as either a Rack endpoint or a Rack middleware.

```ruby
# Example of using RackStack as a middleware (app built using an ordinary Rack::Builder)
Rack::Builder.new {

  use SomeMiddleware

  # RackStack can be used as a middleware, alongside your existing Rack components
  use RackStack.new when: { host: "foo.com" } do
    use AnotherMiddleware
    run SomeApplication.new
  end

  # Or RackStack can be used as a Rack endpoint.
  run RackStack.new do
    run AnotherApplication.new  
  end

}.to_app
```

Stack Manipulation
------------------

A `RackStack` may be manipulated at runtime.

```ruby
rack_stack = RackStack.new

# You can run the RackStack or something else, eg. mount it with Artifice
some_rack_server.run(rack_stack)

# #use, #run, and #map may be used at runtime
rack_stack.use :my_middleware, SomeMiddleware
rack_stack.run SomeApp.new, when: { host: "someapp.com" }
rack_stack.map "/foo" do
  use FooMiddleware
  run FooApp.new
end

# You can easily remove named applications
rack_stack.remove :my_middleware
```

If you want to "void the warranty", feel free to add/remove components to the internal `RackStack#stack`.

`RackStack#stack` is an `Array` of RackStack components, each representing a #use, #map, or #run statement.

```ruby
# For example, if you want to put a #use statement *first*, you can:
rack_stack.stack.unshift RackStack.use(:my_middleware, SomeMiddleware)

# Note that #use/map/run statements are manually reproduced by passing the 
# same arguments to the RackStack::use/map/run methods.

# As another example, you may want to remove un-named applications from your RackStack.
rack_stack.stack.reject! do |component|

  # you can ask a component if it is a use? map? or run?
  if component.use?

    # .instance may be called on any component to get the object that RackStack#get returns for a component.
    instance = component.instance

    # sample of a check that we might want to do to remove a component.
    true if instance.is_a? MyMiddleware && instance.my_middleware_method?
  end
end
```

## Contributing

 1. Fork it
 1. Create your feature branch (`git checkout -b my-new-feature`)
 1. Commit your changes (`git commit -am 'Added some feature'`)
 1. Push to the branch (`git push origin my-new-feature`)
 1. Create new Pull Request

```sh
# This will run all of the tests (RackStack tests, Rack::Builder tests, sample use cases tests)
make test
```

## License

RackStack is released under the MIT license.


[Rack::Builder]: http://rack.rubyforge.org/doc/classes/Rack/Builder.html
[compatibility]: https://github.com/remi/rack-stack/tree/master/spec/rack-builder-compatibility
[artifice]: https://github.com/wycats/artifice
