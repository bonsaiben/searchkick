# Searchkick

:rocket: Intelligent search made easy

Searchkick learns what **your users** are looking for.  As more people search, it gets smarter and the results get better.  It’s friendly for developers - and magical for your users.

Searchkick handles:

- stemming - `tomatoes` matches `tomato`
- special characters - `jalapeno` matches `jalapeño`
- extra whitespace - `dishwasher` matches `dish washer`
- misspellings - `zuchini` matches `zucchini`
- custom synonyms - `qtip` matches `cotton swab`

Plus:

- query like SQL - no need to learn a new query language
- reindex without downtime
- easily personalize results for each user
- autocomplete
- “Did you mean” suggestions
- works with ActiveRecord and Mongoid

:speech_balloon: Get [handcrafted updates](http://chartkick.us7.list-manage.com/subscribe?u=952c861f99eb43084e0a49f98&id=6ea6541e8e&group[0][4]=true) for new features

:tangerine: Battle-tested at [Instacart](https://www.instacart.com)

[![Build Status](https://travis-ci.org/ankane/searchkick.png?branch=master)](https://travis-ci.org/ankane/searchkick)

We highly recommend tracking queries and conversions

:zap: [Searchjoy](https://github.com/ankane/searchjoy) makes it easy

## Get Started

[Install Elasticsearch](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/setup.html). For Homebrew, use:

```sh
brew install elasticsearch
```

Add this line to your application’s Gemfile:

```ruby
gem 'searchkick'
```

For Elasticsearch 0.90, use version `0.6.3` and [this readme](https://github.com/ankane/searchkick/blob/v0.6.3/README.md).

Add searchkick to models you want to search.

```ruby
class Product < ActiveRecord::Base
  searchkick
end
```

Add data to the search index.

```ruby
Product.reindex
```

And to query, use:

```ruby
products = Product.search "2% Milk"
products.each do |product|
  puts product.name
end
```

Searchkick supports the complete [Elasticsearch Search API](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-search.html). As your search becomes more advanced, we recommend you use the [Elasticsearch DSL](#advanced) for maximum flexibility.

### Queries

Query like SQL

```ruby
Product.search "2% Milk", where: {in_stock: true}, limit: 10, offset: 50
```

Search specific fields

```ruby
fields: [:name, :brand]
```

Where

```ruby
where: {
  expires_at: {gt: Time.now}, # lt, gte, lte also available
  orders_count: 1..10,        # equivalent to {gte: 1, lte: 10}
  aisle_id: [25, 30],         # in
  store_id: {not: 2},         # not
  aisle_id: {not: [25, 30]},  # not in
  user_ids: {all: [1, 3]},    # all elements in array
  or: [
    [{in_stock: true}, {backordered: true}]
  ]
}
```

Order

```ruby
order: {_score: :desc} # most relevant first - default
```

[All of these sort options are supported](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-request-sort.html)

Limit / offset

```ruby
limit: 20, offset: 40
```

### Boosting

Boost important fields

```ruby
fields: ["title^10", "description"]
```

Boost by the value of a field (field must be numeric)

```ruby
boost_by: [:orders_count] # give popular documents a little boost
boost_by: {orders_count: {factor: 10}} # default factor is 1
```

Boost matching documents

```ruby
boost_where: {user_id: 1}
boost_where: {user_id: {value: 1, factor: 100}} # default factor is 1000
boost_where: {user_id: [{value: 1, factor: 100}, {value: 2, factor: 200}]}
```

[Conversions](#keep-getting-better) are also a great way to boost.

### Get Everything

Use a `*` for the query.

```ruby
Product.search "*"
```

### Pagination

Plays nicely with kaminari and will_paginate.

```ruby
# controller
@products = Product.search "milk", page: params[:page], per_page: 20
```

View with kaminari

```erb
<%= paginate @products %>
```

View with will_paginate

```erb
<%= will_paginate @products %>
```

### Partial Matches

By default, results must match all words in the query.

```ruby
Product.search "fresh honey" # fresh AND honey
```

To change this, use:

```ruby
Product.search "fresh honey", operator: "or" # fresh OR honey
```

By default, results must match the entire word - `back` will not match `backpack`. You can change this behavior with:

```ruby
class Product < ActiveRecord::Base
  searchkick word_start: [:name]
end
```

And to search (after you reindex):

```ruby
Product.search "back", fields: [{name: :word_start}]
```

Available options are:

```ruby
:word # default
:word_start
:word_middle
:word_end
:text_start
:text_middle
:text_end
```

To boost fields, use:

```ruby
fields: [{"name^2" => :word_start}] # better interface on the way
```

### Exact Matches

```ruby
User.search "hi@searchkick.org", fields: [{email: :exact}, :name]
```

### Language

Searchkick defaults to English for stemming.  To change this, use:

```ruby
class Product < ActiveRecord::Base
  searchkick language: "German"
end
```

[See the list of languages](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/analysis-snowball-tokenfilter.html)

### Synonyms

```ruby
class Product < ActiveRecord::Base
  searchkick synonyms: [["scallion", "green onion"], ["qtip", "cotton swab"]]
end
```

Call `Product.reindex` after changing synonyms.

### WordNet

Prepopulate English synonyms with the [WordNet database](http://en.wikipedia.org/wiki/WordNet).

Download [WordNet 3.0](http://wordnetcode.princeton.edu/3.0/WNprolog-3.0.tar.gz) to each Elasticsearch server and move `wn_s.pl` to the `/var/lib` directory.

```sh
cd /tmp
curl -o wordnet.tar.gz http://wordnetcode.princeton.edu/3.0/WNprolog-3.0.tar.gz
tar -zxvf wordnet.tar.gz
mv prolog/wn_s.pl /var/lib
```

Tell each model to use it:

```ruby
class Product < ActiveRecord::Base
  searchkick wordnet: true
end
```

### Misspellings

By default, Searchkick handles misspelled queries by returning results with an [edit distance](http://en.wikipedia.org/wiki/Levenshtein_distance) of one. To turn off this feature, use:

```ruby
Product.search "zuchini", misspellings: false # no zucchini
```

You can also change the edit distance with:

```ruby
Product.search "zucini", misspellings: {edit_distance: 2} # zucchini
```

### Indexing

Control what data is indexed with the `search_data` method. Call `Product.reindex` after changing this method.

```ruby
class Product < ActiveRecord::Base
  def search_data
    as_json only: [:name, :active]
    # or equivalently
    {
      name: name,
      active: active
    }
  end
end
```

Searchkick uses `find_in_batches` to import documents.  To eager load associations, use the `search_import` scope.

```ruby
class Product < ActiveRecord::Base
  scope :search_import, -> { includes(:searches) }
end
```

By default, all records are indexed.  To control which records are indexed, use the `should_index?` method.

```ruby
class Product < ActiveRecord::Base
  def should_index?
    active # only index active records
  end
end
```

### To Reindex, or Not to Reindex

#### Reindex

- when you install or upgrade searchkick
- change the `search_data` method
- change the `searchkick` method

#### No need to reindex

- App starts

### Stay Synced

There are three strategies for keeping the index synced with your database.

1. Immediate (default)

  Anytime a record is inserted, updated, or deleted

2. Asynchronous

  Use background jobs for better performance

  ```ruby
  class Product < ActiveRecord::Base
    searchkick callbacks: :async
  end
  ```

  And [install Active Job](https://github.com/ankane/activejob_backport) for Rails 4.1 and below

3. Manual

  Turn off automatic syncing

  ```ruby
  class Product < ActiveRecord::Base
    searchkick callbacks: false
  end
  ```

#### Associations

Data is **not** automatically synced when an association is updated.  If this is desired, add a callback to reindex:

```ruby
class Image < ActiveRecord::Base
  belongs_to :product

  after_commit :reindex_product

  def reindex_product
    product.reindex # or reindex_async
  end
end
```

### Keep Getting Better

Searchkick uses conversion data to learn what users are looking for.  If a user searches for “ice cream” and adds Ben & Jerry’s Chunky Monkey to the cart (our conversion metric at Instacart), that item gets a little more weight for similar searches.

The first step is to define your conversion metric and start tracking conversions.  The database works well for low volume, but feel free to use Redis or another datastore.

```ruby
class Search < ActiveRecord::Base
  belongs_to :product
  # fields: id, query, searched_at, converted_at, product_id
end
```

You do **not** need to clean up the search queries.  Searchkick automatically treats `apple` and `APPLES` the same.

Next, add conversions to the index.

```ruby
class Product < ActiveRecord::Base
  has_many :searches

  searchkick conversions: "conversions" # name of field

  def search_data
    {
      name: name,
      conversions: searches.group("query").count
      # {"ice cream" => 234, "chocolate" => 67, "cream" => 2}
    }
  end
end
```

Reindex and set up a cron job to add new conversions daily.

```ruby
rake searchkick:reindex CLASS=Product
```

### Personalized Results

Order results differently for each user.  For example, show a user’s previously purchased products before other results.

```ruby
class Product < ActiveRecord::Base

  def search_data
    {
      name: name,
      orderer_ids: orders.pluck(:user_id) # boost this product for these users
    }
  end

end
```

Reindex and search with:

```ruby
Product.search "milk", boost_where: {orderer_ids: current_user.id}
```

### Autocomplete

Autocomplete predicts what a user will type, making the search experience faster and easier.

![Autocomplete](http://ankane.github.io/searchkick/autocomplete.png)

**Note:** If you only have a few thousand records, don’t use Searchkick for autocomplete. It’s *much* faster to load all records into JavaScript and autocomplete there (eliminates network requests).

First, specify which fields use this feature.  This is necessary since autocomplete can increase the index size significantly, but don’t worry - this gives you blazing faster queries.

```ruby
class City < ActiveRecord::Base
  searchkick text_start: [:name]
end
```

Reindex and search with:

```ruby
City.search "san fr", fields: [{name: :text_start}]
```

Typically, you want to use a JavaScript library like [typeahead.js](http://twitter.github.io/typeahead.js/) or [jQuery UI](http://jqueryui.com/autocomplete/).

#### Here’s how to make it work with Rails

First, add a route and controller action.

```ruby
# app/controllers/cities_controller.rb
class CitiesController < ApplicationController

  def autocomplete
    render json: City.search(params[:query], fields: [{name: :text_start}], limit: 10).map(&:name)
  end

end
```

Then add the search box and JavaScript code to a view.

```html
<input type="text" id="query" name="query" />

<script src="jquery.js"></script>
<script src="typeahead.js"></script>
<script>
  $("#query").typeahead({
    name: "city",
    remote: "/cities/autocomplete?query=%QUERY"
  });
</script>
```

### Suggestions

![Suggest](http://ankane.github.io/searchkick/recursion.png)

```ruby
class Product < ActiveRecord::Base
  searchkick suggest: ["name"] # fields to generate suggestions
end
```

Reindex and search with:

```ruby
products = Product.search "peantu butta", suggest: true
products.suggestions # ["peanut butter"]
```

### Facets

[Facets](http://www.elasticsearch.org/guide/reference/api/search/facets/) provide aggregated search data.

![Facets](http://ankane.github.io/searchkick/facets.png)

```ruby
products = Product.search "chuck taylor", facets: [:product_type, :gender, :brand]
p products.facets
```

By default, `where` conditions are not applied to facets (for backward compatibility).

```ruby
Product.search "wingtips", where: {color: "brandy"}, facets: [:size]
# facets *not* filtered by color :(
```

Change this with:

```ruby
Product.search "wingtips", where: {color: "brandy"}, facets: [:size], smart_facets: true
```

or set `where` conditions for each facet separately:

```ruby
Product.search "wingtips", facets: {size: {where: {color: "brandy"}}}
```

Limit

```ruby
Product.search "2% Milk", facets: {store_id: {limit: 10}}
```

Ranges

```ruby
price_ranges = [{to: 20}, {from: 20, to: 50}, {from: 50}]
Product.search "*", facets: {price: {ranges: price_ranges}}
```

Use the `stats` option to get to max, min, mean, and total scores for each facet

```ruby
Product.search "*", facets: {store_id: {stats: true}}
```

### Highlight

Specify which fields to index with highlighting.

```ruby
class Product < ActiveRecord::Base
  searchkick highlight: [:name]
end
```

Highlight the search query in the results.

```ruby
bands = Band.search "cinema", fields: [:name], highlight: true
```

**Note:** The `fields` option is required, unless highlight options are given - see below.

View the highlighted fields with:

```ruby
bands.with_details.each do |band, details|
  puts details[:highlight][:name] # "Two Door <em>Cinema</em> Club"
end
```

To change the tag, use:

```ruby
Band.search "cinema", fields: [:name], highlight: {tag: "<strong>"}
```

To highlight and search different fields, use:

```ruby
Band.search "cinema", fields: [:name], highlight: {fields: [:description]}
```

Additional options, including fragment size, can be specified for each field:

```ruby
Band.search "cinema", fields: [:name], highlight: {fields: {name: {fragment_size: 200}}}
```

You can find available highlight options in the [Elasticsearch reference](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-request-highlighting.html#_highlighted_fragments).

### Similar Items

Find similar items.

```ruby
product = Product.first
product.similar(fields: ["name"])
```

### Geospatial Searches

```ruby
class City < ActiveRecord::Base
  searchkick locations: ["location"]

  def search_data
    attributes.merge location: [latitude, longitude]
  end
end
```

Reindex and search with:

```ruby
City.search "san", where: {location: {near: [37, -114], within: "100mi"}} # or 160km
```

Bounded by a box

```ruby
City.search "san", where: {location: {top_left: [38, -123], bottom_right: [37, -122]}}
```

### Boost By Distance

Boost results by distance - closer results are boosted more

```ruby
City.search "san", boost_by_distance: {field: :location, origin: [37, -122]}
```

Also supports [additional options](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html#_decay_functions)

```ruby
City.search "san", boost_by_distance: {field: :location, origin: [37, -122], function: :linear, scale: "30mi", decay: 0.5}
```

## Inheritance

Searchkick supports single table inheritance.

```ruby
class Dog < Animal
end
```

The parent and child model can both reindex.

```ruby
Animal.reindex
Dog.reindex # equivalent
```

And to search, use:

```ruby
Animal.search "*"                   # all animals
Dog.search "*"                      # just dogs
Animal.search "*", type: [Dog, Cat] # just cats and dogs
```

**Note:** The `suggest` option retrieves suggestions from the parent at the moment.

```ruby
Dog.search "airbudd", suggest: true # suggestions for all animals
```

## Debugging Queries

See how Elasticsearch tokenizes your queries with:

```ruby
Product.searchkick_index.tokens("Dish Washer Soap", analyzer: "default_index")
# ["dish", "dishwash", "washer", "washersoap", "soap"]

Product.searchkick_index.tokens("dishwasher soap", analyzer: "searchkick_search")
# ["dishwashersoap"] - no match

Product.searchkick_index.tokens("dishwasher soap", analyzer: "searchkick_search2")
# ["dishwash", "soap"] - match!!
```

Partial matches

```ruby
Product.searchkick_index.tokens("San Diego", analyzer: "searchkick_word_start_index")
# ["s", "sa", "san", "d", "di", "die", "dieg", "diego"]

Product.searchkick_index.tokens("dieg", analyzer: "searchkick_word_search")
# ["dieg"] - match!!
```

See the [complete list of analyzers](lib/searchkick/index.rb#L209).

## Deployment

Searchkick uses `ENV["ELASTICSEARCH_URL"]` for the Elasticsearch server.  This defaults to `http://localhost:9200`.

### Heroku

Choose an add-on: [SearchBox](https://addons.heroku.com/searchbox), [Bonsai](https://addons.heroku.com/bonsai), or [Found](https://addons.heroku.com/foundelasticsearch).

```sh
# SearchBox
heroku addons:add searchbox:starter
heroku config:add ELASTICSEARCH_URL=`heroku config:get SEARCHBOX_URL`

# Bonsai
heroku addons:add bonsai
heroku config:add ELASTICSEARCH_URL=`heroku config:get BONSAI_URL`

# Found
heroku addons:add foundelasticsearch
heroku config:add ELASTICSEARCH_URL=`heroku config:get FOUNDELASTICSEARCH_URL`
```

Then deploy and reindex:

```sh
heroku run rake searchkick:reindex CLASS=Product
```

### Other

Create an initializer `config/initializers/elasticsearch.rb` with:

```ruby
ENV["ELASTICSEARCH_URL"] = "http://username:password@api.searchbox.io"
```

Then deploy and reindex:

```sh
rake searchkick:reindex CLASS=Product
```

### Performance

For the best performance, add [Typhoeus](https://github.com/typhoeus/typhoeus) to your Gemfile.

```ruby
gem 'typhoeus'
```

And create an initializer with:

```ruby
require "typhoeus/adapters/faraday"
Ethon.logger = Logger.new("/dev/null")
```

**Note:** Typhoeus is not available for Windows.



