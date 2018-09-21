# GQLi - GraphQL Client for humans

GQLi is a DSL (Domain Specific Language) to consume GraphQL APIs.

## Installation

Install it via the command line:

```bash
gem install gqli
```

Or add it to your `Gemfile`:

```ruby
gem 'gqli'
```

## Usage

### Creating a GraphQL Client

For the examples throught this README, we'll be using the Contentful and Github GraphQL APIs.
Therefore, here's the initialization code required for both of them:

```ruby
require 'gqli'

# Creating a Contentful GraphQL Client
SPACE_ID = 'cfexampleapi'
CF_ACCESS_TOKEN = 'b4c0n73n7fu1'
CONTENTFUL_GQL = GQLi::Client.new(
  "https://graphql.contentful.com/content/v1/spaces/#{SPACE_ID}",
  headers: { "Authorization" => "Bearer #{CF_ACCESS_TOKEN}" }
)

# Creating a Github GraphQL Client
GITHUB_ACCESS_TOKEN = ENV['GITHUB_TOKEN']
GITHUB_GQL = GQLi::Client.new(
  "https://api.github.com/graphql",
  headers: { "Authorization" => "Bearer #{GITHUB_ACCESS_TOKEN}" }
)
```

### Creating a Query

Queries are the way we have to request data from a GraphQL API.
This gem provides a simple DSL to create your own queries.

The query operator is `GQLi::DSL.query`.

```ruby
# Query to fetch the usernames for the first 10 watchers of the first 10 repositories I belong to
WatchersQuery = GQLi::DSL.query {
  viewer {
    login
    repositories(first: 10) {
      edges {
        node {
          nameWithOwner
          watchers(first: 10) {
            edges {
              node {
                login
              }
            }
          }
        }
      }
    }
  }
}
```

### Divide and conquer - using Fragments

Some times, we want to reuse parts of queries.
For that, we can split chunks of our queries into Fragments.

The fragment operator is `GQLi::DSL.fragment`.

To include fragments within other nodes, use the `___` operator as shown below.

To do type matching, use the `__on` operator as shown below.

```ruby
# Base fragment that will be reused for all Cat queries.
CatBase = GQLi::DSL.fragment('CatBase', 'Cat') {
  name
  likes
  lives
}

CatBestFriend = GQLi::DSL.fragment('CatBestFriend', 'Cat') {
  bestFriend {
    # Here, because `bestFriend` is polimorphic in our GraphQL API,
    # we need to explicitly state for which Type we want to include our fragment.
    # To do a type match, instead of GraphQLs `... on SomeType` we do `__on('SomeType')`.
    __on('Cat') {
      # To include a fragment, instead of GraphQLs `...`, we use `___`.
      ___ CatBase
    }
  }
}

# A fragment reusing multiple fragments
CatImportantFields = GQLi::DSL.fragment('CatImportantFields', 'Cat') {
  ___ CatBase
  ___ CatBestFriend
}

# A fragment used to define a query, alongside other regular fields.
CatQuery = GQLi::DSL.query {
  catCollection(limit: 1) {
    items {
      ___ CatImportantFields
      image {
        url
      }
    }
  }
}
```

### Executing the queries

To execute the queries, you need to pass a `Query` object to the client's `#execute` method.
This will return a `Response` object which contains the `data` and the `query` executed.

For example:

```ruby
response = CONTENTFUL_GQL.execute(CatQuery)

puts "Query sent:"
puts response.query.to_gql

puts
puts "Response received"
response.data.catCollection.items.each do |c|
  puts "Name:    #{c.nam}e"
  puts "Likes:   #{c.likes.join(", ")}"
  puts "Lives #: #{c.lives}"
  c.bestFriend.tap do |bf|
    puts "Best Friend:"
    puts "\tName:    #{bf.name}"
    puts "\tLikes:   #{bf.likes.join(", ")}"
    puts "\tLives #: #{bf.lives}"
  end
end
```

This will output:

```
Query sent:
query {
  catCollection(limit: 1) {
    items {
      name
      likes
      lives
      bestFriend {
        ... on Cat {
          name
          likes
          lives
        }
      }
      image {
        url
      }
    }
  }
}

Response received
Name:    e
Likes:   cheezburger
Lives #: 1
Best Friend:
	Name:    Nyan Cat
	Likes:   rainbows, fish
	Lives #: 1337
```

### Embedding the DSL in your classes

If you want to avoid the need for prepending all GQLi DSL's calls with `GQLi::DSL.`, then you can `extend` and/or `include` the module within your own classes.
When using `extend`, you will have access to the DSL at a class level. When using `include` you will have access to the DSL at an object level.

```ruby
class ContentfulClient
  extend GQLi::DSL # Makes DSL available at a class level
  include GQLi::DSL # Makes DSL available at an object level

  SPACE_ID = 'cfexampleapi'
  ACCESS_TOKEN = 'b4c0n73n7fu1'
  CONTENTFUL_GQL = GQLi::Client.new(
    "https://graphql.contentful.com/content/v1/spaces/#{SPACE_ID}",
    headers: { "Authorization" => "Bearer #{ACCESS_TOKEN}" }
  )

  CatBase = fragment('CatBase', 'Cat') {
    name
    likes
    lives
  }

  CatBestFriend = fragment('CatBestFriend', 'Cat') {
    bestFriend {
      __on('Cat') {
        ___ CatBase
      }
    }
  }

  CatImportantFields = fragment('CatImportantFields', 'Cat') {
    ___ CatBase
    ___ CatBestFriend
  }

  def cats(limit)
    CONTENTFUL_GQL.execute(
      query {
        catCollection(limit: limit) {
          items {
            ___ CatImportantFields
            image {
              url
            }
          }
        }
      }
    )
  end
end

response = ContentfulClient.new.cats(5)
```
