# MongodbLogger [![Build Status](https://secure.travis-ci.org/le0pard/mongodb_logger.png)](http://travis-ci.org/le0pard/mongodb_logger)

MongodbLogger is a alternative logger for Rails 3, which log all requests on you application into MongoDB database. 
It:

* simple to integrate into existing Rails 3 application;
* allow store all logs from cluster into one scalable storage - MongoDB;
* flexible schema of MongoDB allow to store and search any information;
* web panel allow filter logs, build graphs using MapReduce by information from your logs;

## Installation

1. Add the following to your Gemfile then refresh your dependencies by executing "bundle install" (or just simple "bundle"):

        gem "mongodb_logger"

1. Add the following line to your ApplicationController:

        include MongodbLogger::Base

1. Add MongodbLogger settings to database.yml for each environment in which you want to use the MongodbLogger. The MongodbLogger will also
   look for a separate mongodb\_logger.yml or mongoid.yml (if you are using mongoid) before looking in database.yml.
   In the mongodb\_logger.yml and mongoid.yml case, the settings should be defined without the 'mongodb_logger' subkey.

   database.yml:

        development:
          adapter: postgresql
          database: my_app_development
          username: postgres
          mongodb_logger:
            database: my_app               # required (the only required setting)
            capsize: <%= 10.megabytes %>   # default: 250MB
            host: localhost                # default: localhost
            port: 27017                    # default: 27017
            replica_set: true              # default: false - Adds retries for ConnectionFailure during voting for replica set master
            safe_insert: false             # default: false - Enable/Disable safe inserts (wait for insert to propagate to all nodes)
            application_name: my_app       # default: Rails.application
            disable_file_logging: false    # default: false - disable logging into filesystem (only in MongoDB)
            collection: some_name          # default: Rails.env + "_log" - name of MongoDB collection

    mongodb_logger.yml:

        development:
          database: my_app
          capsize: <%= 10.megabytes %>
          host: localhost
          port: 27017
          replica_set: true

1. To setup web interface (optional), just mount MongodbLogger::Server in rails routes:
    
    you\_rails\_app/config/routes.rb
        
        mount MongodbLogger::Server.new, :at => "/mongodb"
        
  Now you can see web interface by url "http://localhost:3000/mongodb"
  
  
## Usage

  After success instalation of gem, a new MongoDB document (record) will be created for each request on your application,
  by default will record the following information: Runtime, IP Address, Request Time, Controller, Method, 
  Action, Params, Application Name and All messages sent to the logger. The structure of the MongoDB document looks like this:

      {
        'action'           : action_name,
        'application_name' : application_name (rails root),
        'controller'       : controller_name,
        'ip'               : ip_address,
        'messages'         : {
                               'info'  : [ ],
                               'debug' : [ ],
                               'error' : [ ],
                               'warn'  : [ ],
                               'fatal' : [ ]
                             },
        'params'           : { },
        'path'             : path,
        'request_time'     : date_of_request,
        'runtime'          : elapsed_execution_time_in_milliseconds,
        'url'              : full_url,
        'method'           : request method (GET, POST, PUT, DELETE, OPTIONS)
      }

  Beyond that, if you want to add extra information to the base of the document (let's say something like user_id on every request that it's available),
  you can just call the Rails.logger.add_metadata method on your logger like so (for example from a before_filter):

      # make sure we're using the MongodbLogger in this environment
      if Rails.logger.respond_to?(:add_metadata)
       Rails.logger.add_metadata(:user_id => @current_user.id)
      end


## Querying via the Rails console

  And now, for a couple quick examples on getting ahold of this log data...
  First, here's how to get a handle on the MongoDB from within a Rails console:

      >> db = Rails.logger.mongo_connection
      => #<Mongo::DB:0x007fdc7c65adc8 @name="monkey_logs_dev" ... >

      >> collection = db[Rails.logger.mongo_collection_name]
      => #<Mongo::Collection:0x007fdc7a4d12b0 @name="development_log" .. >

  Once you've got the collection, you can find all requests for a specific user (with id):

      >> cursor = collection.find(:user_id => '12355')
      => #<Mongo::Cursor:0x1031a3e30 ... >
      >> cursor.count
      => 5

  Find all requests that took more that one second to complete:

      >> collection.find({:runtime => {'$gt' => 1000}}).count
      => 3

  Find all order#show requests with a particular order id (id=order_id):

      >> collection.find({"controller" => "order", "action"=> "show", "params.id" => order_id})

  Find all requests with an exception that contains "RoutingError" in the message or stack trace:

      >> collection.find({"messages.error" => /RoutingError/})
      
  Find all requests with errors:

      >> collection.find({"is_exception" => true})

  Find all requests with a request_date greater than '11/18/2010 22:59:52 GMT'

      >> collection.find({:request_time => {'$gt' => Time.utc(2010, 11, 18, 22, 59, 52)}})

      
      
Copyright (c) 2009-2011 Phil Burrows, CustomInk and Leopard released under the MIT license