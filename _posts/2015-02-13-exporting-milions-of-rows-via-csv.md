---
layout: post
title:  "Exporting milions of rows via CSV"
categories: ruby rails sql csv export
---

# Exporting milions of rows via CSV

Exporting CSV seems like a solved issue in the Ruby on Rails framework and to a
large extent it is. There are large amounts of tutorials available on how to
implement an efficient CSV export solution with support for [http
streaming](http://smsohan.com/blog/2013/05/09/genereating-and-streaming-potentially-large-csv-files-using-ruby-on-rails/).
There are two issues with implementing a CSV export in Rails.

1. It takes a considerable amount of time
2. It takes a considerable amount of memory (if one line of the CSV represents
   multiple models)

The solutions to both of these problems is to do your CSV export in the
database and let Rails only serve the response. We'll be using the Postgresql
support for importing and exporting CSVs.

If you want to use the code discussed here, the base modules are available [here](https://github.com/m1k3/csv_base)

## Controller

Lets start by implementing response streaming the same way we would if we used
Rails to construct the CSV response.

``` ruby
class CsvExportController < ApplicationController
  def index
    respond_to do |format|
      format.csv  { render_csv }
    end
  end

  private

  def selected_items
    # implementation omited
  end

  def csv_lines
    Items::CsvExport.new(selected_items)
  end

  def render_csv
    set_file_headers
    set_streaming_headers
    response.status = 200
    self.response_body = csv_lines
  end

  def set_file_headers
    headers['Content-Type'] = 'text/csv; charset=UTF-16LE'
    headers['Content-disposition'] = 'attachment;'
    headers['Content-disposition'] += " filename=\"#{file_name}.csv\""
  end

  def set_streaming_headers
    headers['X-Accel-Buffering'] = 'no'
    headers["Cache-Control"] ||= "no-cache"
    headers.delete("Content-Length")
  end

  def file_name
    'a_big_export'
  end
end
```

There is a lot going on here, but we won't go into details, because [other
posts](http://smsohan.com/blog/2013/05/09/genereating-and-streaming-potentially-large-csv-files-using-ruby-on-rails/)
cover the topic. The most important thing to grasp is that the controller sets
the response body to the return value of the csv_lines method which is an
object that implements the Enumerable interface. The controller will iterate
over that object streaming each row it gets.

Lets have a look at the Exporter class.

## Exporter

``` ruby
module Items
  class CsvExport
    include CsvBase::CsvBase

    def initialize(items)
      @items = items
    end

    private

    attr_reader :items

    def header
      CSV.generate_line(['a column', 'another column'], col_sep: "\t").to_s
    end

    def export_columns
      ['items.a_column', 'related_class.another_column']
    end

    def export_sql
      items.select(export_columns)
    end
  end
end
```

This class defines the particulars of how to get the data for the export. It
receives an ActiveRecord relation object which it uses to construct an export
sql query. The heavy lifting is done by the CsvBase::CsvBase module.

It is important to keep in mind that the implementation of the exporter class
is dependent on the application and this example is provided only as a template
of how an implementation can look like.

## CsvBase

The CsvBase::CsvBase module is designed to be included in a class and uses
methods defined on that class in its #each template method to provide the
enumerator with data and is application independent. Currently the assumption
is that the exported CSV is going to be opened by Excel in windows. It is
however trivial to abstract that asumption away.

``` ruby
module CsvBase
  module CsvBase
    BOM = "\377\376".force_encoding('UTF-16LE')

    include Enumerable

    def each
      yield bom
      yield encoded(header)

      generate_csv do |row|
        yield encoded(row)
      end
    end

    def header
      ''
    end

    def bom
      ::CsvBase::CsvBase::BOM
    end

    private

    def encoded(string)
      string.encode('UTF-16LE', undef: :replace)
    end

    # WARNING: This will most likely NOT work on jruby!!!
    def generate_csv
      conn = ActiveRecord::Base.connection.raw_connection
      conn.copy_data(export_csv_query) do
        while row = conn.get_copy_data
          yield row.force_encoding('UTF-8')
        end
      end
    end

    def export_csv_query
      %Q{copy (#{export_sql}) to stdout with (FORMAT CSV, DELIMITER '\t', HEADER FALSE, ENCODING 'UTF-8');}
    end
  end
end
```

Since the CsvBase::CsvBase module includes the Enumerable module its each method
will end up getting called by the controller. It consists of three steps:

1. It first yields a BOM (byte order mark) which is important for opening the
   CSV in Excel on windows. It can be overriden in the client CsvExport class if
   its not needed or there needs to be something different.
2. It yields a properly encoded header row. The client provides the header.
3. It calls to the generate_csv method yielding whatever that method yields.

The generate_csv method is the interesting part. It uses a low (level
API)[http://deveiate.org/code/pg/PG/Connection.html] of the postgres
connnector. The connector receives a (copy
command)[http://www.postgresql.org/docs/9.2/static/sql-copy.html] with a sql
query constructed to select all rows with a custom delimiter. Since the row
returned from Postgres is a csv string all Ruby has to do is properly encode
the string and pass it along.

That is all there is to it. The download will now be super-fast and consume a
constant amount of memory while generating the output.

For comparison, lets create a new exporter that uses standard find_each approach.

``` ruby
module Items
  class CsvExport
    include Csv::CsvBase

    def initialize(items)
      @items = items
    end

    private

    attr_reader :items

    def header
      CSV.generate_line(['a column', 'another column'], col_sep: "\t").to_s
    end

    def generate_csv
      items.find_each do |item|
        CSV.generate_line([item.a_column, item.association.another_column], col_sep: "\t").to_s
      end
    end
  end
end
```

Exporting 500k records:

1. find_each approach
  * Time: ~450s
  * Peak Memory: ~1.8GB
2. raw_connection approach
  * Time: ~90s
  * Peak memory: ~400MB

## Conclusion

When the middleman is cut out there is an observable 5x speed/memory
performance improvement without without any change in behavior.  Starting out
with this kind of a solution is not recommended because there are a few caveats
to consider when using this kind of solution.

1. We are tying ourselves to postgres by using the details about the
   connection. It is not a problem in most apps, but it is a thing to consider.
2. Problems on non-MRI environments - the raw_connection code will
   most likely cause issues on JRuby.
3. Code is more complicated - the example is factored in such a way that the
   details on how to do build the content of the CSV are encapsulated in a
   single method, but its still an additional burden on the developers
   maintaing that code.

It is therefore preferable to start with a simple approach and only use this
kind of a solution when Rails becomes a real bottleneck.
