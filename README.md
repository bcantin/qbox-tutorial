Welcome to the qbox.io Tutorial
===============================
qbox.io is a service that provides a nice and easy way to use 
ElasticSearch without you, the developer, having to worry about
setting up ES in your production environment.  It is recommended
that you do setup ES on your machine for development, but that is
beyond the scope of this tutorial.  We will use qbox.io as our
development to get a feel for what we can do.

First things first!  We need a qbox.io API key.  If you have not already done so, 
head over to [qbox.io](http://qbox.io), sign up and get your API key.

Once you have your api key, we can test that it is working by using curl from the
terminal.

```shell
curl http://api.qbox.io/YOUR-API-KEY
```

create an index, and see if it exists.

```shell
curl -XPUT http://api.qbox.io/YOUR-API-KEY/test_index
# wait for a return
curl http://api.qbox.io/YOUR-API-KEY/test_index
```

We can also view that our test_index was created from the [Dashboard](http://qbox.io/dashboard/indexes) that qbox.io provides. 

We also can delete the index and verify that as well.

```shell
curl -XDELETE http://api.qbox.io/YOUR-API-KEY/test_index
```

Let's create a test_index, add some items, and verify that we can get them from qbox

```shell
curl -XPUT http://api.qbox.io/YOUR-API-KEY/test_index
curl -XPOST http://api.qbox.io/YOUR-API-KEY/test_index/product -d '{
  "name" : "Product Name",
  "price" : 24.99
}'
# add an item with a custom ID
curl -XPUT http://api.qbox.io/YOUR-API-KEY/test_index/product/my_custom_id -d '{
  "name" : "Product Name 2",
  "price" : 20.99
}'
# get our item with the custom id
curl http://api.qbox.io/YOUR-API-KEY/test_index/product/my_custom_id

# and let's destroy this index
curl -XDELETE http://api.qbox.io/YOUR-API-KEY/test_index
```

Using curl is nice, as most languages support some type of curl library. qbox.io
provices some examples of their API using Ruby, and with that knowledge we can
take an existing Ruby on Rails application and add some search capabilities.

Let's get comfortable using just Ruby and qbox.io first.  I am using Ruby 2.0.0, but 1.9.x should be compatable as well.

As their Ruby API overview shows us, we can use the Ruby Tire library to interface
with qbox.io.  Tire provides a great way to use ElasticSearch, and for the majority
of the qbox.io calls, it sends our request onto ElasticSearch without interfering
with it.  

Since we want to use some data, we can use the faker library as well to create
some data and put it into our qbox.io index.  We will need to get both the tire 
and faker gems on our sytem if they are not there already.

```shell
gem install tire faker
```

Fire up IRB and require both tire and faker

```ruby
require 'tire'
require 'faker'
```

Set up Tire to connect to qbox.io.  Tire will use this url for all requests until we change it.

```ruby
Tire.configure do
  url "http://api.qbox.io/YOUR-API-KEY"
end
```

Add an index and ensure that it exists after creation
```ruby
Tire.index "test_index" do
  create
end
# wait for confirmation
Tire.index "test_index" do
  exists?
end
```

Add an item and then get it back.
```ruby
Tire.index "test_index" do
  store :type => "test", :id => 42, :name => "Dog skateboard", :price => 2599.99
end

Tire.index "test_index" do
  retrieve "test", 42
end
```
We can also destroy and item in our index if we know the id

```ruby
Tire.index "test_index" do
  remove :type => "test", :id => 42
end
```

We have access to bulk importing
```ruby
tests = [
  { :id => "1", :type => "test", :name => "Left Boot",    :tags => ["cowboys"]               },
  { :id => "2", :type => "test", :name => "Squirrel Nut", :tags => ["rodents", "nut lovers"] },
  { :id => "3", :type => "test", :name => "Beach Ball",   :tags => ["hippies", "babies"]     },
  { :id => "4", :type => "test", :name => "Dynamite",     :tags => ["misc", "not baby safe"] }
]

Tire.index "test_index" do
  import tests
end
```

Now let's perform some searching.

```ruby
search = Tire.search "test_index" do
  query do
    string "name:B*"
  end

  filter :terms, :tags => ["cowboys"]

  sort { by :id, "desc" }

  facet "global-tags", :global => true do
    terms :tags
  end

  facet "current-tags" do
    terms :tags
  end
end

search.results.each do |document|
  puts "* #{ document.name } [tags: #{ document.tags.join(', ') }]"
end

# => * Left Boot [tags: cowboys]
# => [<Item id: "1", type: "test", name: "Left Boot", tags: ["cowboys"], _score: nil, _type: "test", _index: "brad.cantin-test_index", _version: nil, sort: [1], highlight: nil, _explanation: nil>] 

# all tags outside of our search
search.results.facets["global-tags"]["terms"].each do |f|
  puts "#{ f['term'] } - #{ f['count'] }"
end
# => safe - 1
# => rodents - 1
# => nut - 1
# => misc - 1
# => lovers - 1
# => hippies - 1
# => cowboys - 1
# => baby - 1
# => babies - 1
# => [{"term"=>"safe", "count"=>1}, {"term"=>"rodents", "count"=>1}, {"term"=>"nut", "count"=>1}, {"term"=>"misc", "count"=>1}, {"term"=>"lovers", "count"=>1}, {"term"=>"hippies", "count"=>1}, {"term"=>"cowboys", "count"=>1}, {"term"=>"baby", "count"=>1}, {"term"=>"babies", "count"=>1}] 
# all tags that are returned in our search terms
search.results.facets["current-tags"]["terms"].each do |f|
  puts "#{ f['term'] } - #{ f['count'] }"
end

# => hippies - 1
# => cowboys - 1
# => babies - 1
```
Now let's add information that is similiar to our samples Rails application.  In our 
Rails app we sell hats.  Our hats have various styles and various colors.  Our
customers would like to search on a term and also narrow down the results by 
colors and by style.

We can create 500 hat objects and put them into qbox.io.
Let's destroy our test_index and create a store_index and add 500 hats.

```ruby
Tire.index('test_index').delete
Tire.index('store_index').create
Tire.index("store_index").exists?

colors = %w{brown black white grey tan blue red green orange}
hat_style  = %w{baseball beanie beret derby cricket deerstalker 
                dunce fedora fez patrol sombereo stetson top zucchetto}
hats  = []
500.times do |id|
  hats << {type:   "hat", 
           id:     id,
           name:   Faker::Lorem.words(rand(3)+1).join(" "),
           price:  rand(100) + 0.99,
           colors: colors.sample(rand(2)+1),
           style:  hat_style.sample(rand(3)+1) }
end
```

We can bulk import our hats as we did above with products.
```ruby
Tire.index "store_index" do
  import hats
end
```

We can get a hat out of our index and compare it to a hat in our array.  Your results
may not look like mine below, but it should be similiar

```ruby
request = Tire.index "store_index" do
            retrieve "hat", 30
          end
item = JSON.parse(request.response.body)["_source"]
# => => {"type"=>"hat", "id"=>30, "name"=>"nulla", "price"=>36.99, "colors"=>["grey"], "style"=>["baseball", "derby", "top"]} 

hats[30]
 # => {:type=>"hat", :id=>30, :name=>"nulla", :price=>36.99, :colors=>["grey"], :style=>["baseball", "derby", "top"]} 
```

Both objects are very similiar, the only difference is that we get back JSON from
qbox.io and the keys are strings instead of symbols. 

Data looks good.  search time!
```ruby
search = Tire.search "store_index" do
  query do
    string "name:rerum"
  end
end
# => #<Tire::Search::Search:0x007fc73e042d30 @indices=["store_index"], @types=[], @options={}, @path="/store_index/_search", @query=#<Tire::Search::Query:0x007fc73e0426f0 @value={:query_string=>{:query=>"name:rerum"}}>> 

search.results.each do |document|
  puts "#{document.id} #{ document.name } {#{document.type}} [colors: #{ document.colors.join(', ') }] price: #{document.price} [style: #{ document.style.join(', ') }]"
end

# => 14 rerum {hat} [colors: tan, white] price: 48.99 [style: patrol, deerstalker]
# => 264 rerum {hat} [colors: grey, tan] price: 98.99 [style: fez, fedora]
# => 15 rerum {hat} [colors: white] price: 89.99 [style: sombereo, fez, fedora]
# => 459 rerum {hat} [colors: white, red] price: 49.99 [style: deerstalker]
# => 298 rerum {hat} [colors: black, blue] price: 93.99 [style: sombereo, top]
# => 445 rerum commodi {hat} [colors: brown] price: 37.99 [style: cricket, dunce]
# => 168 magni rerum {hat} [colors: red, green] price: 61.99 [style: baseball, fedora, sombereo]
# => 296 rerum ipsa est {hat} [colors: blue, white] price: 87.99 [style: beret]
# => 107 rerum rem doloremque {hat} [colors: black] price: 44.99 [style: fedora, beanie, deerstalker]
# =>  435 sit porro rerum {hat} [colors: grey, blue] price: 4.99 [style: cricket, sombereo, beanie]
```

Our search returned 10 results, but we could have more than 10.  By default 
ElasticSearch returns 10 items at a time.
To see how many total results we have access to, Tire has a convience method ```:total```
Again, your results will vary.

```ruby
search.results.total # => 14
```

To get the last 4 items, we can update our query and perform our search again. Tire
provides a handy way to do this ```perform```

```ruby
search.from(10)
search.perform.results.each do |document|
  puts "#{document.id} #{ document.name } {#{document.type}} [colors: #{ document.colors.join(', ') }] price: #{document.price} style: [#{ document.style.join(', ') }]"
end

# => 411 rerum eum vitae {hat} [colors: orange, white] price: 39.99 style: [patrol, deerstalker, stetson]
# => 78 rerum ut numquam {hat} [colors: tan, brown] price: 66.99 style: [cricket]
# => 276 rerum impedit possimus {hat} [colors: grey] price: 0.99 style: [deerstalker, fedora]
# => 379 ut exercitationem rerum {hat} [colors: red, tan] price: 85.99 style: [beret, fedora, baseball]
```


We can also change the default number of items returned from our search.  Let's reset
our ```from``` to 0 and change the ```size``` to 30.  Again, your search may have more or
less than 30 total.  Experiment with different query strings.
```ruby
search.size(30)
search.from(0)
search.perform.results.each do |document|
  puts "#{document.id} #{ document.name } {#{document.type}} [colors: #{ document.colors.join(', ') }] price: #{document.price} style: [#{ document.style.join(', ') }]"
end
# => 14 rerum {hat} [colors: tan, white] price: 48.99 style: [patrol, deerstalker]
# => 264 rerum {hat} [colors: grey, tan] price: 98.99 style: [fez, fedora]
# => 15 rerum {hat} [colors: white] price: 89.99 style: [sombereo, fez, fedora]
# => 459 rerum {hat} [colors: white, red] price: 49.99 style: [deerstalker]
# => 298 rerum {hat} [colors: black, blue] price: 93.99 style: [sombereo, top]
# => 445 rerum commodi {hat} [colors: brown] price: 37.99 style: [cricket, dunce]
# => 168 magni rerum {hat} [colors: red, green] price: 61.99 style: [baseball, fedora, sombereo]
# => 296 rerum ipsa est {hat} [colors: blue, white] price: 87.99 style: [beret]
# => 107 rerum rem doloremque {hat} [colors: black] price: 44.99 style: [fedora, beanie, deerstalker]
# => 435 sit porro rerum {hat} [colors: grey, blue] price: 4.99 style: [cricket, sombereo, beanie]
# => 411 rerum eum vitae {hat} [colors: orange, white] price: 39.99 style: [patrol, deerstalker, stetson]
# => 78 rerum ut numquam {hat} [colors: tan, brown] price: 66.99 style: [cricket]
# => 276 rerum impedit possimus {hat} [colors: grey] price: 0.99 style: [deerstalker, fedora]
# => 379 ut exercitationem rerum {hat} [colors: red, tan] price: 85.99 style: [beret, fedora, baseball]
```

Now lets experiment some more with only getting hats that are blue. Create a new
search object and get the results. As before, your results will vary from mine.

```ruby
search = Tire.search "store_index" do
  query do
    terms :colors, ["blue"]
  end
end

search.results.each do |document|
  puts "#{document.id} #{ document.name } {#{document.type}} [colors: #{ document.colors.join(', ') }] price: #{document.price} style: [#{ document.style.join(', ') }]"
end
# => 42 est quo {hat} [colors: blue] price: 31.99 style: [cricket, zucchetto, deerstalker]
# => 176 totam labore {hat} [colors: blue] price: 28.99 style: [top]
# => 253 perspiciatis pariatur {hat} [colors: blue] price: 92.99 style: [sombereo]
# => 53 sint totam saepe {hat} [colors: blue] price: 93.99 style: [derby, fez, top]
# => 93 quo et voluptatem {hat} [colors: blue] price: 64.99 style: [derby, zucchetto]
# => 134 itaque molestiae {hat} [colors: blue] price: 54.99 style: [top]
# => 295 voluptatem ab est {hat} [colors: blue] price: 61.99 style: [stetson, sombereo]
# => 327 sint et {hat} [colors: blue] price: 17.99 style: [fez, derby]
# => 349 odit sit {hat} [colors: blue] price: 11.99 style: [fez, fedora]
# => 484 rem sint {hat} [colors: blue] price: 11.99 style: [top]

search.results.total # => 78
```

Alter our search to use both the ```string 'name:rerum'``` and our color filter

```ruby
search = Tire.search "store_index" do
  query do
    string "name:rerum"
  end
  filter :terms, :colors => ["blue"]
end

search.results.each do |document|
  puts "#{document.id} #{ document.name } {#{document.type}} [colors: #{ document.colors.join(', ') }] price: #{document.price} style: [#{ document.style.join(', ') }]"
end
# => 298 rerum {hat} [colors: black, blue] price: 93.99 style: [sombereo, top]
# => 296 rerum ipsa est {hat} [colors: blue, white] price: 87.99 style: [beret]
# => 435 sit porro rerum {hat} [colors: grey, blue] price: 4.99 style: [cricket, sombereo, beanie]
search.results.total # => 3 
```

Filters are a powerful way to narrow down queries.  Update our query to get the same
hats, but either blue or black
```ruby
search = Tire.search "store_index" do
  query do
    string "name:rerum"
  end
  filter :terms, :colors => ["black", "blue"]
end
search.results.each do |document|
  puts "#{document.id} #{ document.name } {#{document.type}} [colors: #{ document.colors.join(', ') }] price: #{document.price} style: [#{ document.style.join(', ') }]"
end
# => 298 rerum {hat} [colors: black, blue] price: 93.99 style: [sombereo, top]
# => 296 rerum ipsa est {hat} [colors: blue, white] price: 87.99 style: [beret]
# => 107 rerum rem doloremque {hat} [colors: black] price: 44.99 style: [fedora, beanie, deerstalker]
# => 435 sit porro rerum {hat} [colors: grey, blue] price: 4.99 style: [cricket, sombereo, beanie]
search.results.total # => 4
```
By default ElasticSearch combines filters to use OR.  Consult the the ElasticSearch
documentation to see how to change this to an AND.

We can also add facets.  Facets are a way to get an insight into your data with
the current search results. Let's add a color facet.  As before, your results
will vary from below.
```ruby
search = Tire.search "store_index" do
  query do
    string "name:rerum"
  end
  filter :terms, :colors => ["black", "blue"]
  facet "current-colors" do
    terms :colors
  end  
end

search.results.facets["current-colors"]["terms"].each do |f|
  puts "#{ f['term'] } - #{ f['count'] }"
end
# => white - 5
# => tan - 4
# => red - 3
# => grey - 3
# => blue - 3
# => brown - 2
# => black - 2
# => orange - 1
# => green - 1
```
We can also add a global facet.  This can help show how many total results our
index has globally, as in not using our search queries or filters.
```ruby
search = Tire.search "store_index" do
  query do
    string "name:rerum"
  end  

  facet "global-style", :global => true do
    terms :style
  end
end

search.results.facets["global-style"]["terms"].each do |f|
  puts "#{ f['term'] } - #{ f['count'] }"
end
# => derby - 87
# => stetson - 84
# => fez - 74
# => zucchetto - 61
# => fedora - 60
# => patrol - 57
# => beanie - 57
# => dunce - 56
# => cricket - 54
# => sombereo - 53
```
As with the item results, by default we only get back the top 10 facets.

We should have enough information to now add qbox.io to our Rails application.

Destroy the index we just created before moving on.

```ruby
Tire.index('test_index').delete
```

A sample application, Madhatter, is provided at [this repository](https://github.com/bcantin/madhatter)
The master branch is the finished application.  There is a before_search branch
that you can use to follow along and add the functionality with this tutorial.

```shell
git clone https://github.com/bcantin/madhatter.git
cd madhatter
# get the before_search branch
git fetch origin before_search:before_search
git checkout before_search
```

There are four main files that we will be updating

* app/models/hat.rb
* app/controllers/hats_controller.rb
* app/views/hats/index.html.haml
* app/views/shared/_sidebar.html.haml

Our application uses mysql as the database.  Please update the config/database.yml
to your machine.
Once that is done, we can get started.  We need to run a few commands to get
ready.

```shell
bundle install
rake db:setup 
rails server
```

Open up your browser to localhost:3000 and you should see our basic application.
Nothing fancy, no searches and static first 10 hats.

Our first step is to connect to our application to qbox.io.  We can accomplish
this through the Tire library.

Update our Gemfile

```ruby
# ...
gem 'tire'
# ...
```

And run bundle install
```shell
bundle install
```

Add our qbox.io API key to an initializer 

Create:config/initializers/qbox_io.rb

```ruby
# configure tire to connect to qbox.io
Tire.configure do
  url "http://api.qbox.io/YOUR-API-KEY"
end

## NOTE: In a production application, we would want the configuration to 
#        point to another ElasticSearch installation so we would not 
#        impact our production data when adding features in development or
#        when running tests
```
Add Tire to our Hat model

Update app/models/hat.rb
```ruby
class Hat < ActiveRecord::Base
  include Tire::Model::Search
  include Tire::Model::Callbacks
  # ...
end
```
Now let's jump into the rails console and see what we can do.
```shell
rails console
```
```ruby
hat = Hat.first
#  Hat Load (0.5ms)  SELECT `hats`.* FROM `hats` LIMIT 1
=> #<Hat id: 1, name: "laborum vel qui", description: "Fugiat beatae ea quas assumenda est.", price: #<BigDecimal:7fb4a5212788,'0.7799E2',18(18)>, created_at: "2013-04-27 12:33:53", updated_at: "2013-04-27 12:33:53"> 
hat.name
# => 
hat.colors
  ActsAsTaggableOn::Tag Load (0.5ms)  SELECT `tags`.* FROM `tags` INNER JOIN `taggings` ON `tags`.`id` = `taggings`.`tag_id` WHERE `taggings`.`taggable_id` = 1 AND `taggings`.`taggable_type` = 'Hat' AND (taggings.context = 'colors')
 => [#<ActsAsTaggableOn::Tag id: 1, name: "blue">, #<ActsAsTaggableOn::Tag id: 2, name: "tan">, #<ActsAsTaggableOn::Tag id: 3, name: "brown">, #<ActsAsTaggableOn::Tag id: 4, name: "grey">] 
hat.color_list
 => ["blue", "tan", "brown", "grey"] 

# perform a search
hats = Hat.search("vel")
[REQUEST FAILED] curl -X GET 'http://api.qbox.io/xlworctl/hats/hat/_search?size=10&pretty' -d '{"query":{"query_string":{"query":"vel"}},"size":10}'
Tire::Search::SearchRequestFailed: 404 : Your account has no index named hats.
rails 
```

Well, that was diappointing, but then again, we do not want index to be added until
we as the developer create them.  So back in our console we can create our index.
And if we read the failure message, we can see that by default it is trying to
use an index named 'hats'.
```ruby
Tire.index('hats').create
# perform our seach again
hats = Hat.search("vel")
hats.results # => []
```

No results?  Well if we think about it, all we did was add the index. We didn't
populate our qbox.io index with any data yet.  And if we can remember, the
Tire library has a nice way to import bulk data.  Back in our console, let's do it.

```ruby
Hat.import
# After a few seconds...
hats = Hat.search("vel")
hats.results.count #=> 10
```

Good, we have results. As before, the default number returned by ElasticSeach is 10.
And as before, Tire has a convience method to see how many total we have access to
with this search.

```ruby
hats.total # => 34
# look at our first results hat
hats.results.first
# => <Item (Hat) created_at: "2013-04-27T12:34:14Z", description: "Autem aliquid.", name: "vel consequatur", price: "40.99", updated_at: "2013-04-27T12:34:14Z", id: "452", _score: 0.93521, _type: "hat", _index: "brad.cantin-hats", _version: nil, sort: nil, highlight: nil, _explanation: nil> 
```
We have some information, but it is not all the information we would like about a
hat.  We want to see the list of colors and the list of styles as well. We also
know that we want those to be searchable.

To accomplish this, we need to tell tire to add those to our index.  There are two
ways to do this. One is to use the mapping block syntax, and the other way is
to implement a ```:to_indexed_json``` method. Since that is the easier of the two, we
can do that quickly.

Edit app/models/hat.rb

```ruby
class Hat < ActiveRecord::Base
  # ...
  def to_indexed_json
    { name: name,
      description: description,
      price: price,
      color_list: color_list,
      style_list: style_list}.to_json
  end
  # ...
end
```

Back in the conosle we need to reload the rails application and reindex our data.

```ruby
reload!
Hat.import
```

During this import, we should have seen all of the colors and styles being loaded from
the database and imported into our index on qbox.io.  Let's perform our search again.

```ruby
hats = Hat.search("vel")
hats.results.count # => 10
hats.total # => 34
# same as before for the count and total as expected. 
# Look at our first result and we get
hats.results.first
# => <Item (Hat) name: "nam dolores non", description: "Vel et voluptate et.", price: "26.99", color_list: ["brown", "grey", "orange"], style_list: ["sombereo", "fez", "beret", "baseball", "cricket"], id: "289", _score: 0.93521, _type: "hat", _index: "brad.cantin-hats", _version: nil, sort: nil, highlight: nil, _explanation: nil> 
```

Much better!  We now can see that the style_list and color_list is being indexed.

Now let's update our controller and add some real searching!  If you had closed it,
start up a rails server in another terminal.

```shell
rails server
```

We are using Twitter-Bootstrap for our layout.  We can make it presentable while we
make it functional.

Edit app/assets/stylesheets/hats.css.scss
```css
#main { 
  padding-top: 15px; 
}
```

Edit app/views/hats/index.html.haml and put the following at the top of the file
```haml
= form_tag hats_path, :method => 'get', id: 'search', class: "form-search" do 
  = text_field_tag :search, params[:search], class: "span3"
  = submit_tag 'Search', class: 'btn' 
```

Edit app/controllers/hats_controller.rb
```ruby
class HatsController < ApplicationController

  def index
    @hats = Hat.search(params[:search])
  end

end
```
Now refresh our main page and ... crash!
ElasticSearch doesn't accept nil being passed in for our search query.
A quick update...

Edit app/controllers/hats_controller.rb
```ruby
class HatsController < ApplicationController

  def index
    @hats = Hat.search(params[:search] || 'name:""')
  end

end
```

That works, but we got 0 results.  I think it would be better if we got items
back if there isn't a search query.  While we are at it, we should move the
searching in the Hat model.

Edit app/controllers/hats_controller.rb
```ruby
class HatsController < ApplicationController

  def index
    @hats = Hat.searchable(params)
  end

end
```

Edit app/models/hat.rb
```ruby
class Hat < ActiveRecord::Base
  # ...
  def self.searchable(opts={})
    if opts[:search].present?
      @hats = Hat.search(opts[:search], page: opts[:page])
    else
      @hats = Hat.page(opts[:page]).limit(10)
    end
    @hats
  end
  # ...
end
```

Now perform a blank search and we get back some hats.  Perform
a search on 'foo' and we should get no items.  Performing a
search on 'vel' should return some items.

As we can also see, we have no pagination.  We are using the kaminari gem
and it can help us with some nice convience pagination methods.  There is
also a nice generator for us since we are using bootstrap.  From a terminal
run the following generator.

```shell
rails g kaminari:views bootstrap -e haml
```

We can add the pagination links into our view.

Edit app/views/hats/index.html.haml
```haml
= form_tag hats_path, :method => 'get', id: 'search', class: "form-search" do 
  = text_field_tag :search, params[:search], class: "span3"
  = submit_tag 'Search', class: 'btn'

.row
  = render "/shared/sidebar"

  .span8
    = paginate @hats
    = render @hats
```

Back in our browser, refresh our main page and see what it looks like.  Click on
a few of the pagination links and see that it works.

Now we want to update our sidebar so we can include some filters for styles
and colors.

We can use the rails console to experiment.  Build a base hash to pass in and ensure we get results.

```ruby
hash = {search: 'vel'}
results = Hat.searchable(hash)
results.total # => 34
```
Good, we have a baseline of 34 items.  Add a filter on styles and it should narrow
down the number of results we get back.
```ruby
hash[:style] = ['beret']
```
Using the Tire DSL we can create a search:
```ruby
results = Hat.search do
            query { string hash[:search] }
            filter :terms, style_list: hash[:style] 
          end
results.total # => 4
```
4 results is much less than 34, so we narrowed down the total.  Remove the 
hash[:style] and rerun the block.
```ruby
hash[:style] = nil
results = Hat.search do
            query { string hash[:search] }
            filter :terms, style_list: hash[:style] 
          end
```

As before, ES doesn't like nil or blank parameters to be passed in.  Let's fix that.
```ruby
results = Hat.search do
            query { string hash[:search] }
            filter :terms, style_list: hash[:style] if hash[:style].present?
          end
```


Update our Hat model searchable method to use a block

Edit app/models/hat.rb
```ruby
class Hat < ActiveRecord::Base
  # ...
  def self.searchable(opts={})
    if opts[:search].present?
      @hats = Hat.search page: opts[:page] || 1 do
        query { string opts[:search] }
        filter(:terms, style_list: opts[:style]) if opts[:style].present?
      end
    else
      @hats = Hat.page(opts[:page] || 1).limit(10)
    end
    @hats
  end
  # ...
end
```

Adding checkboxes to the sidebar will allow our customers to narrow down the results
so they can choose what they want while searching.

Let's do styles first.  We would like an array of styles passed in that we can use in
the hat model as a filter. We need to initialize a style array in our hats controller

Edit app/controllers/hats_controller.rb
```ruby
class HatsController < ApplicationController

  def index
    params[:style] ||= []
    @hats = Hat.searchable(params)
  end

end
```
We would like the checkbox and the label to both be clickable.

Edit app/views/shared/_sidebar.html.haml
```haml
.span2
  #sidebar
    %h3 Styles
    %ul
      - Hat::STYLES.each do |style|
        %li
          %label
            = check_box_tag 'style[]', style, params[:style].include?(style)
            = style

    %h3 Colors
    %ul
      - Hat::COLORS.each do |color|
        %li= color
```

Update the list element style and add a dotted line next to the sidebar as well

Edit app/assets/stylesheets/hats.css.scss
```css
#main { 
  padding-top: 15px; 
}

#sidebar {
  border-right: 1px dotted #ccc;
}

ul li {
  list-style-type: none;
}
```

Refresh our browser and we should see some checkboxes next to our styles.
Click on few and submit.  Looking at the logs we do not see the styles
array being passed into the parameters.  We need to update our view
so the sidebar is nested within our haml form.

Edit app/views/hats/index.html.haml
```haml
= form_tag hats_path, :method => 'get', id: 'search', class: "form-search" do 
  = text_field_tag :search, params[:search], class: "span3"
  = submit_tag 'Search', class: 'btn'

  .row
    = render "/shared/sidebar"

    .span8
      = paginate @hats
      = render @hats
```

We should update our _hat partial as well to ensure we are getting the
proper results

Edit app/views/hats/_hat.html.haml
```haml
%p
  = hat.name
  = number_to_currency(hat.price)
  = "[styles: #{hat.style_list}]"
  = "[colors: #{hat.color_list}]"
```

Ensure that only 'beret' is checked and that we are searching on 'vel'. 
Click on Search and our view should only have 4 results. Our logfile has
a parameter line like below
```ruby
Parameters: {"utf8"=>"✓", "search"=>"vel", "commit"=>"Search", "style"=>["beret"]}
```

Feel free to perform more searches.

If you tried any blank searches with some of the sidebar items checked, you probably
noticed that our search does not properly work.  Let's fix that. Update our 
Hat.searchable method to use a block.  We can also remove the ```if``` check 
for the opts[:search] at the beginning of the method as well.

Edit app/models/hat.rb
```ruby
class Hat < ActiveRecord::Base
  # ...
  def self.searchable(opts={})
    @hats = Hat.search(page: opts[:page] || 1) do
      query { string opts[:search] }           if opts[:search].present?
      filter(:terms, style_list: opts[:style]) if opts[:style].present?
    end
  end
  # ...
end
```

Now let's add some color. Jump back into the rails console.

```ruby
hash = {color: ["brown"]}

results = Hat.search page: hash[:page] || 1 do
  query { string hash[:search] } if hash[:search].present?
  filter :terms, style_list: hash[:style] if hash[:style].present?
  filter :terms, color_list: hash[:color] if hash[:color].present?  
end

results.total # => 150

# add in a seach term

hash[:search] = 'vel'

results = Hat.search page: hash[:page] || 1 do
  query { string hash[:search] } if hash[:search].present?
  filter :terms, style_list: hash[:style] if hash[:style].present?
  filter :terms, color_list: hash[:color] if hash[:color].present?  
end
results.total # => 13
# add in our style as well
hash[:style] = ["beret"]

results = Hat.search page: hash[:page] || 1 do
  query { string hash[:search] } if hash[:search].present?
  filter :terms, style_list: hash[:style] if hash[:style].present?
  filter :terms, color_list: hash[:color] if hash[:color].present?  
end
results.total #=> 2
```

Looking good.  Whe had our style of 'beret' and our search on 'vel' we got 4
results. Adding in our color of 'brown' narrowed it down even more
so. Now we should update our Hat model, sidebar, and controller.

Edit app/models/hat.rb
```ruby
class Hat < ActiveRecord::Base
  # ...
  def self.searchable(opts={})
    @hats = Hat.search(page: opts[:page] || 1) do
      query { string opts[:search] }           if opts[:search].present?
      filter(:terms, style_list: opts[:style]) if opts[:style].present?
      filter(:terms, color_list: opts[:color]) if opts[:color].present?
    end
  end
  # ...
end
```
Edit app/views/shared/_sidebar.html.haml
```haml
.span2
  #sidebar
    %h3 Styles
    %ul
      - Hat::STYLES.each do |style|
        %li
          %label
            = check_box_tag 'style[]', style, params[:style].include?(style)
            = style

    %h3 Colors
    %ul
      - Hat::COLORS.each do |color|
        %li
          %label
            = check_box_tag 'color[]', color, params[:color].include?(color)
            = color
```

Edit app/controllers/hats_controller.rb
```ruby
class HatsController < ApplicationController

  def index
    params[:style] ||= []
    params[:color] ||= []
    @hats = Hat.searchable(params)
  end

end
```

Refresh our page and we should see some results.  Click on a style of only
'beret', a color of only 'brown' and search on 'vel'.  We should get
only two results.

Looking good.  Now we can add some facets.

ElasticSearch allows us to add a facet and a global facet to our searches.
Both have their uses.  A facet is useful for showing how our
search parameters narrow down the items.  A global facet can show us
the total results outside of our searching.

It will be up to you to decide which type of facet best fits your 
application. We will add both types of facets to get and idea
of how to do so.

Also, remember that adding facets, does add some extra query time in
ElasticSearch.

To see both in action, we will add a facet to styles and a global facet
to colors.  Back to the console! Let's use a facet on the styles first.

```ruby
hash = {:search=>"vel"} 

results = Hat.search  do
  query { string hash[:search] }
  facet "current-styles" do
    terms :style_list
  end
end

# as before, we get the usual number of results
results.total # => 34

# check our facets
results.facets["current-styles"]["terms"]
# => {"_type"=>"terms", "missing"=>0, "total"=>101, "other"=>22, "terms"=>[{"term"=>"sombereo", "count"=>12}, {"term"=>"top", "count"=>9}, {"term"=>"patrol", "count"=>9}, {"term"=>"beanie", "count"=>9}, {"term"=>"zucchetto", "count"=>7}, {"term"=>"stetson", "count"=>7}, {"term"=>"fez", "count"=>7}, {"term"=>"derby", "count"=>7}, {"term"=>"dunce", "count"=>6}, {"term"=>"baseball", "count"=>6}]} 
```

Use a nice way to display them in the console

```ruby
results.facets["current-styles"]["terms"].each do |f|
  puts "#{ f['term'] } - #{ f['count'] }"
end

#=> sombereo - 12
#=> top - 9
#=> patrol - 9
#=> beanie - 9
#=> zucchetto - 7
#=> stetson - 7
#=> fez - 7
#=> derby - 7
#=> dunce - 6
#=> baseball - 6
```

We are only getting 10 facets, but we should be getting 14, as we have 14 styles.  Looking
at the [ElasticSearch documentation](http://www.elasticsearch.org/guide/reference/api/search/facets/), 
we can see that we need to pass in a size parameter.

Update our query.

```ruby
results = Hat.search  do
  query { string hash[:search] }
  facet "current-styles" do
    terms :style_list, size: 14
  end
end

# display them nicely
results.facets["current-styles"]["terms"].each do |f|
  puts "#{ f['term'] } - #{ f['count'] }"
end

#=> sombereo - 12
#=> top - 9
#=> patrol - 9
#=> beanie - 9
#=> derby - 8
#=> zucchetto - 7
#=> stetson - 7
#=> fez - 7
#=> baseball - 7
#=> fedora - 6
#=> dunce - 6
#=> deerstalker - 6
#=> cricket - 4
```

Let's alter that result to be a hash of the style to the count
```ruby
facets = {}
results.facets["current-styles"]["terms"].each {|f| facets[f['term']] = f['count']}

# => {"sombereo"=>12, "top"=>9, "patrol"=>9, "beanie"=>9, "derby"=>8, 
#    "zucchetto"=>7, "stetson"=>7, "fez"=>7, "baseball"=>7, "fedora"=>6, 
#    "dunce"=>6, "deerstalker"=>6, "cricket"=>4} 

# get an individual result
facets['cricket']
 => 4 
```
It should be easy to add this into our Hat.searchable method and the view now.

Edit app/models/hat.rb
```ruby
class Hat < ActiveRecord::Base
  # ...
  def self.searchable(opts={})
    @hats = Hat.search(page: opts[:page] || 1) do
      query { string opts[:search] }           if opts[:search].present?
      filter(:terms, style_list: opts[:style]) if opts[:style].present?
      filter(:terms, color_list: opts[:color]) if opts[:color].present?
      facet "current-styles" do
        terms :style_list, size: STYLES.size
      end        
    end
  end
  # ...
end
```
Edit app/controllers/hats_controller.rb
```ruby
class HatsController < ApplicationController

  def index
    params[:style] ||= []
    params[:color] ||= []
    @hats = Hat.searchable(params)

    @hat_styles = {}
    @hats.facets["current-styles"]["terms"].each {|f| @hat_styles[f['term']] = f['count']}
  end

end
```

Edit app/views/shared/_sidebar.html.haml
```haml
.span2
  #sidebar
    %h3 Styles
    %ul
      - Hat::STYLES.each do |style|
        %li
          %label
            = check_box_tag 'style[]', style, params[:style].include?(style)
            = style
            = "(#{@hat_styles[style].to_i})"
    %h3 Colors
    %ul
      - Hat::COLORS.each do |color|
        %li
          %label
            = check_box_tag 'color[]', color, params[:color].include?(color)
            = color
```


Now if we add a search term of 'vel' we can see the sidebar numbers are vastly
different than if we do not provide a search term.

Now, let's add a global facet for the colors.  Jump into the console first.
```ruby
hash = {:search=>"vel"} 

results = Hat.search  do
  query { string hash[:search] }
  facet "current-styles" do
    terms :style_list, count: 14
  end
  facet "global-colors", global: true do
    terms :color_list
  end
end

# as usual, we get the same total
results.total # => 34

# look at our global facet
results.facets["global-colors"]['terms']
# =>  [{"term"=>"grey", "count"=>154}, {"term"=>"red", "count"=>152}, 
#      {"term"=>"brown", "count"=>150}, {"term"=>"white", "count"=>146}, 
#      {"term"=>"blue", "count"=>137}, {"term"=>"black", "count"=>136}, 
#      {"term"=>"tan", "count"=>135}, {"term"=>"orange", "count"=>131}, 
#      {"term"=>"green", "count"=>109}] 
```

And as before, we can display a nicer list view

```ruby
results.facets["global-colors"]["terms"].each do |f|
  puts "#{ f['term'] } - #{ f['count'] }"
end

# => grey - 154
# => red - 152
# => brown - 150
# => white - 146
# => blue - 137
# => black - 136
# => tan - 135
# => orange - 131
# => green - 109
```
We can use the same logic as with the ```current-styles``` facet
to update our model, controller, and view

Edit app/models/hat.rb
```ruby
class Hat < ActiveRecord::Base
  # ...
  def self.searchable(opts={})
    @hats = Hat.search(page: opts[:page] || 1) do
      query { string opts[:search] }           if opts[:search].present?
      filter(:terms, style_list: opts[:style]) if opts[:style].present?
      filter(:terms, color_list: opts[:color]) if opts[:color].present?
      facet "current-styles" do
        terms :style_list, size: STYLES.size
      end        
    end
  end
  # ...
end
```

Edit app/controllers/hats_controller.rb
```ruby
class HatsController < ApplicationController

  def index
    params[:style] ||= []
    params[:color] ||= []
    @hats = Hat.searchable(params)

    @hat_styles = {}
    @hats.facets["current-styles"]["terms"].each {|f| @hat_styles[f['term']] = f['count']}

    @hat_colors = {}
    @hats.facets["global-colors"]["terms"].each {|f| @hat_colors[f['term']] = f['count']}
  end

end
```
Edit update app/views/shared/_sidebar.html.haml
```haml
.span2
  #sidebar
    %h3 Styles
    %ul
      - Hat::STYLES.each do |style|
        %li
          %label
            = check_box_tag 'style[]', style, params[:style].include?(style)
            = style
            = "(#{@hat_styles[style].to_i})"
    %h3 Colors
    %ul
      - Hat::COLORS.each do |color|
        %li
          %label
            = check_box_tag 'color[]', color, params[:color].include?(color)
            = color
            = "(#{@hat_colors[color].to_i})" 
```

As we can see, the facet results for our styles changes as we alter
our search string, but the facets for our colors does not change.

There are many other things we could add to this application, such as
using ajax whenever we change a parameter on the sidebar.
We could also update our sidebar to be an AND type of search instead of 
the default OR search that ElasticSearch uses.

I encourage you to read through the documentation of Tire, ElasticSearch,
and, of course Qbox.io to get a feel of what you can do

Enjoy!


