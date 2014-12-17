---
layout: post
title: "What happens when you grow a database sideways?"
date: 2014-10-30 17:32:09 -0400
comments: true
categories: 
---

The short answer is that it gets slow. I had heard this before but decided to confirm it with some code. I used SQLite for the database, ActiveRecord to ease the SQL work, and Ruby to automate table generation. Tables of sizes from 100 to 1000 columns and rows were created and populated, and then randomly queried 100 times per benchmark. The benchmarks were averaged and compiled to generate the summary chart.  To keep things simple, I only tested a single column of 1000 records, or a single row with 1000 attributes (columns). 

The code for the helper methods:

```ruby
require 'benchmark'
require "./config/environment.rb"
ActiveRecord::Base.logger = Logger.new("./log")

def destroy_db
    `rm "db/halloween_development.sqlite"`
end

def migrate_db
  `rake db:migrate`
end

def bench_the_block(num, &block)
    Benchmark.bmbm do |x|
      x.report do
        num.times do 
          block.call
        end
      end
    end
end

def create_migrations(num_attribs,  model_name)#model name plural

  if model_name=="candies"
  `rm "db/migrate/01_create_#{model_name}.rb"`
    f=File.open("./db/migrate/01_create_#{model_name}.rb", "w")

  else
  `rm "db/migrate/02_create_#{model_name}.rb"`
    f=File.open("./db/migrate/02_create_#{model_name}.rb", "w")
  end

  table_attributes=""
  (1..num_attribs).each do |num|
    table_attributes += "t.string :'#{num}'\n"
  end

  f.write("""class Create#{model_name.capitalize} < ActiveRecord::Migration
            def change
            create_table :#{model_name} do |t|
              #{table_attributes}
            end
          end
        end
        """)
  f.close

  migrate_db
end

def populate(num_columns, model_name, mode)#model name singular

  m=Class.const_get(model_name.capitalize)
    if mode=="horizontal"
    m.create("1" => "x") #House.create("1" => "x")

    (1..num_columns).each do |num|
      m.find(1).update("#{num}" => "#{num}")
    end

    elsif mode=="vertical"
      (1..num_columns).each do |num|
      m.create("1" => "#{num}")

    end
  end
end



num=1000#number of columns or rows
times=100#number of times to run the benchmark


# destroy_db
# create_migrations(num, "houses")#houses
# populate(num, "house", "horizontal")

# create_migrations(1, "candies")w
# populate(num, "candy", "vertical")

# puts "Random lookups at #{num} columns"
# bench_the_block(times){z=rand(1..num); House.find_by("#{z}"=>z)}

# puts "Random lookups at #{num} rows"
# bench_the_block(times){Candy.find_by("1" => "#{rand(1..num)}")}
```

##Summary
SQLite3 has a default column limit of 2000! So I wasn't able to get to the millions of columns I wanted to try. This can be changed but only to about 35k columns. Cursory review of stack overflow indicates that having a database with more than a 100 columns is usually a design flaw with the database. 

{% img left /images/SQLITE_chart.png 500 400 'image' 'images' %}


