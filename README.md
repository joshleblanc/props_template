# PropsTemplate

PropsTemplate is a direct-to-Oj, JBuilder-like DSL for building JSON. It has support for Russian-Doll caching, layouts, and of course, its most unique feature: your templates are queryable.

[![Build Status](https://circleci.com/gh/thoughtbot/props_template.svg?style=shield)](https://circleci.com/gh/thoughtbot/props_template)

PropsTemplate is fast!

Most libraries would build a hash before feeding it to your serializer of choice, typically Oj. PropsTemplate writes directly to Oj using `Oj::StringWriter` as its rendering your template and skips the need for an intermediate data structure.

PropsTemplate also improves caching. While other libraries spend time unmarshaling, merging, and then serializing to JSON; PropsTemplate simply takes the cached string and [push_json](http://www.ohler.com/oj/doc/Oj/StringWriter.html#push_json-instance_method).


Example:

```ruby
json.flash flash.to_h

json.menu do
  # all keys will be formatted as camelCase

  json.current_user do
    json.email current_user.email
    json.avatar current_user.avatar
    json.inbox current_user.messages.count
  end
end

json.dashboard(defer: :auto) do
  sleep 5
  json.complex_post_metric 500
end

json.posts do
  page_num = params[:page_num]
  paged_posts = @posts.page(page_num).per(20)

  json.list do
    json.array! paged_posts, key: :id do |post|
      json.id post.id
      json.description post.description
      json.comments_count post.comments.count
      json.edit_path edit_post_path(post)
    end
  end

  json.pagination_path posts_path
  json.current paged_posts.current_page
  json.total @posts.count
end


json.footer partial: 'shared/footer' do
end
```

## Installation
If you plan to use PropsTemplate alone just add it to your Gemfile.

```
gem 'props_template'
```

and run `bundle`

## API

### json.set! or json.<your key here>
Defines the attribute or stucture. All keys are automatically camelized lower.

```ruby
json.set! :author_details, {..options...} do
  json.set! :first_name, 'David'
end

or

json.author_details, {..options...} do
  json.first_name, 'David'
end


# => {"authorDetails": { "firstName": "David" }}
```

The inline form defines key and value

| Parameter | Notes |
| :--- | :--- |
| key | A json object key|
| value | A value |

```ruby
json.set! :first_name, 'David'

or

json.first_name 'David'

# => { "firstName": "David" }
```

The block form defines key and structure

| Parameter | Notes |
| :--- | :--- |
| key | A json object key|
| options | Additional [options](#options)|
| block | Additional `json.set!`s or `json.array!`s|

```ruby
json.set! :details do
 ...
end

or

json.details do
 ...
end
```

The difference between the block form and inline form is
1. The block form is an internal node. Partials, Deferement and other [options](#options) are only available on the block form.
2. The inline form is considered a leaf node, and you can only [search](#traversing) for internal nodes.

### json.array!
Generates an array of json objects.

```ruby
collection = [
  {name: 'john'},
  {name: 'jim'}
]

json.details do
  json.array! collection, {....options...} do |person|
    json.first_name person[:name]
  end
end

# => {"details": [
  {"firstName": 'john'},
  {"firstName": 'jim'}
]}
```

| Parameter | Notes |
| :--- | :--- |
| collection | A collection that responds to `member_at` and `member_by` |
| options | Additional [options](#options)|

To support [traversing nodes](react-redux.md#traversing-nodes), any list passed to `array!` MUST implement `member_at(index)` and `member_by(attr, value)`.

For example, if you were using a delegate:

```ruby
class ObjectCollection < SimpleDelegator
  def member_at(index)
    at(index)
  end

  def member_by(attr, val)
    find do |ele|
      ele[attr] == val
    end
  end
end
```

Then in your template:

```ruby
data = ObjectCollection.new([{id: 1, name: 'foo'}, {id: 2, name: 'bar'}])

json.array! data do
  ...
end
```

Similarly for ActiveRecord:

```ruby
class ApplicationRecord < ActiveRecord::Base
  def self.member_at(index)
    offset(index).limit(1).first
  end

  def self.member_by(attr, value)
    find_by(Hash[attr, val])
  end
end
```

Then in your template:

```ruby
json.array! Post.all do
  ...
end
```

#### **Array core extension**

For convenience, PropsTemplate includes a core\_ext that adds these methods to `Array`. For example:

```ruby
require 'props_template/core_ext'
data = [{id: 1, name: 'foo'}, {id: 2, name: 'bar'}]

json.posts
  json.array! data do
    ...
  end
end
```

PropsTemplate does not know what the elements are in your collection. The example above will be fine for [traversing](props-template.md#traversing_nodes) by index `\posts?bzq=posts.0`, but will raise a `NotImplementedError` if you query by attribute `/posts?bzq=posts.id=1`. You may still need a delegate that implements `member_by`.

### json.deferred!
Returns all deferred nodes used by the [deferment](#deferment) option.

```ruby
json.deferred json.deferred!
```

This method is normally used in `application.json.props` when first generated by `rails breezy:install:web`

### json.fragments!
Returns all fragment nodes used by the [partial fragments](#partial-fragments) option.

```ruby
json.fragments json.fragments!
```

This method is normally used in `application.json.props` when first generated by `rails breezy:install:web`

## Options
Functionality such as Partials, Deferements, and Caching can only be set on a block. It is normal to see empty blocks.

```ruby
json.post(partial: 'blog_post') do
end
```

### Partials

Partials are supported. The following will render the file `views/posts/_blog_posts.json.props`, and set a local variable `foo` assigned with @post, which you can use inside the partial.

```ruby
json.one_post partial: ["posts/blog_post", locals: {post: @post}] do
end
```

Usage with arrays:

```ruby
# as an option on an array. The `as:` option is supported when using `array!`
json.posts do
  json.array! @posts, partial: ["posts/blog_post", locals: {foo: 'bar'}, as: 'post'] do
  end
end
```

### Partial Fragments

A fragment uses a digest to identify a rendered partial across your page state in Redux. When BreezyJS recieves a payload with a fragment, it will update every fragment with the same digest in your Redux store.

You would need use partials and add the option `fragment: true`.

```ruby
# index.json.props
json.header partial: ["profile", fragment: true] do
end

# _profile.json.props
json.profile do
  json.address do
    json.state "New York City"
  end
end
```

When using fragments with Arrays, the argument **MUST** be a lamda:

```ruby
require 'props_template/core_ext' #See (lists)[#Lists]

json.array! ['foo', 'bar'], partial: ["footer", fragment: ->(x){ x == 'foo'}]
```

PropsTemplate creates a name for the partial using a digest of your locals, partial name, and globalId (to_json as fallback if there is no globalId) on objects that you pass. You may override this behavior and use a custom identifier:

```ruby
# index.js.breezy
json.header partial: ["profile", fragment: 'me_header'] do
end
```

### Caching
Caching is supported on any node.

Usage:

```ruby
json.author(cache: "some_cache_key") do
  json.first_name "tommy"
end

#or

json.profile(cache: "cachekey", partial: ["profile", locals: {foo: 1}]) do
end

#or nest it

json.author(cache: "some_cache_key") do
  json.address(cache: "some_other_cache_key") do
    json.zip 11214
  end
end
```

When used with arrays, PropsTemplate will use `Rails.cache.read_multi`.

```ruby
require 'props_template/core_ext' #See (lists)[#Lists]

opts = {
  cache: ->(i){ ['a', i] }
}
json.array! [4,5], opts do |x|
  json.top "hello" + x.to_s
end

#or on arrays with partials

opts = {
  cache: (->(d){ ['a', d.id] }),
  partial: ["blog_post", as: :blog_post]
}
json.array! @options, opts
```

### Deferment

You can defer rendering of expensive nodes in your content tree using the `defer: :auto` option. Behind the scenes PropsTemplates will no-op the block entirely, replace the value with `{}` as a placeholder.
When the client recieves the payload, BreezyJS will use the meta data to issue a `remote` dispatch to fetch the missing node and immutibly graft it at the appropriate keypath in your Redux store.

You can access what was deferred with `json.deferred!`. If you use the generators, this will be set up in `application.json.props`.

Usage:

```ruby
json.dashboard(defer: :auto) do
  sleep 10
  json.some_fancy_metric 42
end
```

A manual option is also available:

```ruby
json.dashboard(defer: :manual) do
  sleep 10
  json.some_fancy_metric 42
end
```

Finally in your `application.json.props`:

```ruby
json.defers json.deferred!
```


If `:manual` is used, PropsTemplate will no-op the block and will not populate `json.deferred!`. Its up to you to [query](props-template.md#traversing_nodes) to fetch the node seperately. A common usecase would be tab content that does not load until you click the tab.

#### Working with arrays
The default behavior for deferements is to use the index of the collection to identify an element. PropsTemplate will generate `?_bzq=a.b.c.0.title` in its metadata.

If you wish to use an attribute to identify the element. You must:
1. Implement `:key` to specify which attribute you want to use to uniquely identify the element in the collection. PropsTemplate will generate `?_bzq=a.b.c.some_id=some_value.title`
2. Implement `member_at`, and `member_key` on the collection to allow for BreezyJS to traverse the tree based on key value attributes.

For example:

```ruby
require 'props_template/core_ext' #See (lists)[#Lists]

data = [{id: 1, name: 'foo'}, {id: 2, name: 'bar'}]

json.posts
  json.array! data, key: :some_id do |item|
    json.contact(defer: :auto) do
      json.address '123 example drive'
    end

   # json.some_id item.some_id will be appended automatically to the end of the block
  end
end
```

When BreezyJS receives the response, it will automatically kick off `remote(?bzq=posts.some_id=1.contact)` and `remote(?bzq=posts.some_id=2.contact)`.

# Traversing

PropsTemplate has the ability to walk the tree you build, skipping execution of untargeted nodes. This feature is useful for partial updating your frontend state. See [traversing nodes](react-redux.md#traversing-nodes)

```ruby
traversal_path = ['data', 'details', 'personal']

json.data(search: traversal_path) do
  json.details do
    json.employment do
      ...more stuff...
    end

    json.personal do
      json.name 'james'
      json.zip_code 91210
    end
  end
end

json.footer do
 ...
end
```

PropsTemplate will will walk breath first, finds the matching key, executes the associated block, then repeats until it the node is found. The above will output the below:

```json
{
  data: {
    name: 'james',
    zipCode: 91210
  },
  footer: {
    ....
  }
}
```

Breezy's searching only works with blocks, and will NOT work with Scalars ("leaf" values). For example:

```ruby
traversal_path = ['data', 'details', 'personal', 'name'] <- not found

json.data(search: traversal_path) do
  json.details do
    json.personal do
      json.name 'james'
    end
  end
end

```

## Nodes that do not exist

Nodes that are not found will not define the key where search was enabled on.

```ruby
traversal_path = ['data', 'details', 'does_not_exist']

json.data(search: traversal_path) do
  json.details do
    json.personal do
      json.name 'james'
    end
  end
end

json.footer do
 ...
end

```

The above will render:

```
{
  footer: {
    ...
  }
}
```
