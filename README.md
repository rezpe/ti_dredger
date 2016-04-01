## Overview

TI Dredger provides a set of Web Services to query Impala databases. The main features are:
- Parsing and validating queries
- Starting batch jobs
- Providing status on the jobs
- Providing the results of the job once it is finished

## API

TI dredger uses the api provided by TI SQLLegalize.

## How does it work

TI Dredger Github uses TI SQLLegalize to provide responses to SQL Query to an Impala database.

First it includes the project in the gemfile

```ruby
gem 'ti_sqlegalize', git: 'http://github.com/ebastien/ti_sqlegalize'
```

Then it configures the database to be used by TISqllegalize (in a rb file in the initializers.rb)

```ruby
require 'impala'

label = "#{Rails.env}_hadoop"
config = Rails.configuration.database_configuration[label]

unless config && config['host'] && config['port']
  fail KeyError, "No Impala configuration found for #{label}"
end

module Impala
  class Cursor
    def schema
      metadata.schema.fieldSchemas.map(&:name)
    end

    def parse_row(raw)
      fields = raw.split(metadata.delim)
      metadata.schema.fieldSchemas.zip(fields).map do |schema, raw_value|
        convert_raw_value(raw_value, schema)
      end
    end
  end
end

TiSqlegalize.database = if Rails.env == 'test'
                          -> { TiSqlegalize::DummyDatabase.new }
                        else
                          -> { Impala.connect(config['host'], config['port']) }
                        end
```

And mounts the engine in the routes.rb

```ruby
Rails.application.routes.draw do

  scope Rails.configuration.x.root_path do

    scope module: 'api/v1' do
      get '/', to: 'entry#index', as: :api_v1
      get '/profile', to: redirect('/profile.txt'), as: :api_v1_profile
    end

    mount TiSqlegalize::Engine, at: 'sql'

  end

  match "*path", to: "errors#routing", via: :all
end
```

## Additional Notes

The gemfile in TI Sqlegalize is used to specify the location of tirailsauth as it is not an official gem.
