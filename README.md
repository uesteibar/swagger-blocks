# Swagger::Blocks

[![Build Status](https://travis-ci.org/fotinakis/swagger-blocks.svg?branch=master)](https://travis-ci.org/fotinakis/swagger-blocks)
[![Gem Version](https://badge.fury.io/rb/swagger-blocks.svg)](http://badge.fury.io/rb/swagger-blocks)

Swagger::Blocks is a DSL for pure Ruby code blocks that can be turned into JSON.

It helps you write API docs in the [Swagger](https://helloreverb.com/developers/swagger) style in Ruby and then automatically build JSON that is compatible with [Swagger UI](http://petstore.swagger.wordnik.com/#!/pet).

## Features

* Supports **live updating** by design. Change code, refresh your API docs.
* **Works with all Ruby web frameworks** including Rails, Sinatra, etc.
* **100% support** for all features of the [Swagger 1.2](https://github.com/wordnik/swagger-spec/blob/master/versions/1.2.md) and [Swagger 2.0](https://github.com/swagger-api/swagger-spec/blob/master/versions/2.0.md) specs.
* Flexible—you can use Swagger::Blocks anywhere, split up blocks to fit your style preferences, etc. Since it's pure Ruby and serves definitions dynamically, you can easily use initializers/config objects to change values or even **show different APIs based on environment**.
* 1:1 naming with the Swagger spec—block names and nesting should match almost exactly with the swagger spec, with rare exceptions to make things more convenient.

## Swagger UI demo

http://petstore.swagger.wordnik.com/#!/pet

![swagger-sample](https://cloud.githubusercontent.com/assets/75300/5822830/4769805c-a08c-11e4-9efe-d57cf0f752e0.png)

## Installation

Add this line to your application's Gemfile:

    gem 'swagger-blocks'

Or install directly with `gem install swagger-blocks`.

## Swagger 2.0 example (Rails)

This is a simplified example based on the objects in the Petstore [Swagger Sample App](http://petstore.swagger.wordnik.com/#!/pet). For a more complex and complete example, see the [swagger_v2_blocks_spec.rb](https://github.com/fotinakis/swagger-blocks/blob/master/spec/lib/swagger_v2_blocks_spec.rb) file, or see the [v1.2 docs](https://github.com/fotinakis/swagger-blocks/blob/master/README_v1_2.md).

Also note that **Rails is not required**, you can use Swagger::Blocks in plain Ruby objects.

### PetsController

```Ruby
class PetsController < ActionController::Base
  include Swagger::Blocks

  swagger_path '/pets/{id}' do
    operation :get do
      key :description, 'Returns a single pet if the user has access'
      key :operationId, 'findPetById'
      parameter do
        key :name, :id
        key :in, :path
        key :description, 'ID of pet to fetch'
        key :required, true
        key :type, :integer
        key :format, :int64
      end
      response 200 do
        key :description, 'pet response'
        schema do
          key :'$ref', :Pet
        end
      end
      response :default do
        key :description, 'unexpected error'
        schema do
          key :'$ref', :ErrorModel
        end
      end
    end
  end
  swagger_path '/pets' do
    operation :get do
      key :description, 'Returns all pets from the system that the user has access to'
      key :operationId, 'findPets'
      key :produces, [
        'application/json',
        'text/html',
      ]
      parameter do
        key :name, :tags
        key :in, :query
        key :description, 'tags to filter by'
        key :required, false
        key :type, :array
        items do
          key :type, :string
        end
        key :collectionFormat, :csv
      end
      parameter do
        key :name, :limit
        key :in, :query
        key :description, 'maximum number of results to return'
        key :required, false
        key :type, :integer
        key :format, :int32
      end
      response 200 do
        key :description, 'pet response'
        schema do
          key :type, :array
          items do
            key :'$ref', :Pet
          end
        end
      end
      response :default do
        key :description, 'unexpected error'
        schema do
          key :'$ref', :ErrorModel
        end
      end
    end
    operation :post do
      key :description, 'Creates a new pet in the store.  Duplicates are allowed'
      key :operationId, 'addPet'
      key :produces, [
        'application/json'
      ]
      parameter do
        key :name, :pet
        key :in, :body
        key :description, 'Pet to add to the store'
        key :required, true
        schema do
          key :'$ref', :PetInput
        end
      end
      response 200 do
        key :description, 'pet response'
        schema do
          key :'$ref', :Pet
        end
      end
      response :default do
        key :description, 'unexpected error'
        schema do
          key :'$ref', :ErrorModel
        end
      end
    end
  end

  # ...
end
```

### Models

#### Pet model

```Ruby
class Pet < ActiveRecord::Base
  include Swagger::Blocks

  swagger_schema :Pet do
    key :required, [:id, :name]
    property :id do
      key :type, :integer
      key :format, :int64
    end
    property :name do
      key :type, :string
    end
    property :tag do
      key :type, :string
    end
  end

  swagger_schema :PetInput do
    allOf do
      schema do
        key :'$ref', :Pet
      end
      schema do
        key :required, [:name]
        property :id do
          key :type, :integer
          key :format, :int64
        end
      end
    end
  end

  # ...
end
```

#### Error model

``` Ruby
class ErrorModel  # Notice, this is just a plain ruby object.
  include Swagger::Blocks

  swagger_schema :ErrorModel do
    key :required, [:code, :message]
    property :code do
      key :type, :integer
      key :format, :int32
    end
    property :message do
      key :type, :string
    end
  end
end
```

### Docs controller

To integrate these definitions with Swagger UI, we need a docs controller that can serve the JSON definitions.

```Ruby
resources :apidocs, only: [:index]
```

```Ruby
class ApidocsController < ActionController::Base
  include Swagger::Blocks

  swagger_root do
    key :swagger, '2.0'
    info do
      key :version, '1.0.0'
      key :title, 'Swagger Petstore'
      key :description, 'A sample API that uses a petstore as an example to ' \
                        'demonstrate features in the swagger-2.0 specification'
      key :termsOfService, 'http://helloreverb.com/terms/'
      contact do
        key :name, 'Wordnik API Team'
      end
      license do
        key :name, 'MIT'
      end
    end
    key :host, 'petstore.swagger.wordnik.com'
    key :basePath, '/api'
    key :consumes, ['application/json']
    key :produces, ['application/json']
  end

  # A list of all classes that have swagger_* declarations.
  SWAGGERED_CLASSES = [
    PetsController,
    Pet,
    ErrorModel,
    self,
  ].freeze

  def index
    render json: Swagger::Blocks.build_root_json(SWAGGERED_CLASSES)
  end
end
```

The special part of this controller is this line:

```Ruby
render json: Swagger::Blocks.build_root_json(SWAGGERED_CLASSES)
```

That is the only line necessary to build the full [root Swagger object](https://github.com/swagger-api/swagger-spec/blob/master/versions/2.0.md#schema) JSON and all definitions underneath it. You simply pass in a list of all the "swaggered" classes in your app.

Now, simply point Swagger UI at `/apidocs` and everything should Just Work™. If you change any of the Swagger block definitions, you can simply refresh Swagger UI to see the changes.

### Swagger 1.2 example (Rails)

See the [v1.2 docs](https://github.com/fotinakis/swagger-blocks/blob/master/README_v1_2.md).

## Reference

See the [swagger_v2_blocks_spec.rb](https://github.com/fotinakis/swagger-blocks/blob/master/spec/lib/swagger_v2_blocks_spec.rb) for examples of more complex features and declarations possible.

## Contributing

1. Fork it ( https://github.com/fotinakis/swagger-blocks/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request

## Release notes

* v1.1: Support for Swagger 2.0 spec.
* v1.0.1: Make backwards-compatible with Ruby 1.9.3.
* v1.0.0: Initial major release.

## Credits

Thanks to [@ali-graham](https://github.com/ali-graham) for contributing support for Swagger 2.0.

Original idea inspired by [@richhollis](https://github.com/richhollis/)'s [swagger-docs](https://github.com/richhollis/swagger-docs/) gem.
