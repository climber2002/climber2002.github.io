---
layout: post
title: "PostGIS and Google Maps in Rails Part 1"
date: 2014-05-18 21:53:32 +0800
comments: true
categories: [postgis, google maps, rails]
---

Recently I need to work on a Rails project which is to manage the public and private schools in Gabon, a small country in Africa. And I need to integrate PostGIS and Google maps. I will create a series of blogs to share what I have done, and I will point out some common gotchas while integrate Rails with PostGIS and Google maps. 

Books and Blogs
---------------
To work on PostGIS and Google Maps of course you need to learn them. This series is not a complete tutorial. To study PostGIS, the definite reference is *PostGIS in Action* by Regina O. Obe and Leo S. Hsu. Currently the second edition is in MEAP state and you can buy the ebook at [manning website](http://manning.com/obe2/). To study Google maps, the documentation at [google website](https://developers.google.com/maps/documentation/javascript/tutorial) is already awesome, but if you prefer a book form, the book *Google Maps JavaScript API Cookbook* by Alper Dincer and Balkan Uraz is a good choice, which you can buy the kindle version at [amazon](http://www.amazon.com/Google-Maps-JavaScript-API-Cookbook-ebook/dp/B00HJR6RD6)

In the project to integrate PostGIS with Rails, I utilized some rubygems from RGeo. The original author of those rubygems, Daniel Azuma, has created a series of articles on [his website](http://daniel-azuma.com/articles/georails), which is very helpful. Actually this series has been inspired from his works.

Let's start
-----------
In this first blog I'll describe how to create a rails project and how to create migrations to import GIS data from shape files to our database. 

Install
-------
To work on PostGIS of course you need to install it first. If you work on a Mac OS environment like me, using the [PostgreSQL App](http://postgresapp.com/) is the easiest way. This package already includes PostGIS 2.1, so you just need to enable it, which we will introduce later.

And since we use the [RGeo](https://github.com/rgeo/rgeo) and some of its addons, you need to install [GEOS](http://trac.osgeo.org/geos) and [Proj](http://trac.osgeo.org/proj), which RGeo depends on. For Mac users the easiest way is to install the frameworks [here](http://www.kyngchaos.com/software:frameworks)

Create the application
----------------------
Now let's create a rails application and use PostgreSQL database (I use Rails 4.1).

```
$ rails new schoolpro --database==postgresql
```

Here my application name is schoolpro. Now open the Gemfile and add the *activerecord-postgis-adapter* gem. 

{% codeblock lang:ruby %}
gem 'activerecord-postgis-adapter'
{% endcodeblock %}

And then in the database.yml, let's change the adapter from postgresql to postgis

{% codeblock lang:yml %}
  adapter: postgis
{% endcodeblock %}

Now let's create the database by running **rake db:create**
```
$ rake db:create
```

After the database is created, we need to enable the PostGIS extension by running following command
```
rake db:gis:setup
```

<!-- more -->

Get the shape files
------------------
In this application I need to import some data from shape files to the database. The shapefile is a flat file format for geospatial data originally developed by ESRI for storing sets of geographic features. It supports certain vector shapes-- points, lines, and polygons-- along with associated attributes. Although shapefile began as a proprietary format, the format specification is readily available, and it is now a de facto standard for large datasets. A shapefile actually consists of three (and sometimes more) related files, each with the same base filename but different extensions. The main file has the extension ".shp" and contains the geometric data itself in a binary format. An auxiliary ".shx" file provides a simple flat index allowing random access into the shapefile. A second auxiliary ".dbf" file provides the attribute data in dBASE format. All shapefiles should have those three core files, although some shapefiles may include additional files containing coordinate system, spatial index, or other related information.

The shape files I got is from [gdam](http://www.gadm.org/), which is a spatial database of the locations of the world's administration areas. What I need is the shape files of Gabon. Just go to [the download page](http://www.gadm.org/country), select *Gabon* and File format as *Shapefile* and download it. The downloaded is a zip file, unzip it you will get 3 sets of shapefiles:

* GAB_adm0: This contains country data
* GAB_adm1: This contains provinces data
* GAB_adm2: This contains cities data

Let's create a folder *shpfiles* under #{Rails.root}/db and copy all these files there. These files will be used in our migrations.

Create the country model
------------------------
My goal is to import the country polygon data from a shape file into the database in rails migrations. Now let's import the countries shapefile. Firstly we need to create a **Country** model. So let's create a model **Country** and add following content in the migration file,

{% codeblock lang:ruby %}
class CreateCountries < ActiveRecord::Migration
  def change
    create_table :countries do |t|
      t.string :name, :unique => true
      t.string :iso_code, :unique => true
      t.multi_polygon :geom, :srid => 4326
      t.timestamps
    end

    change_table :countries do |t|
      t.index :geom, :spatial => true
    end
  end
end
{% endcodeblock %}

Here we specified 3 attributes, *name* is country name, the *iso_code* is 3 letter code, for Gabon it's **GAB**. The **t.multi_polygon** is to create a multipolygon geometry column named *geom*, which represents that this column can store multipolygon type in PostGIS. And we set the SRID to 4326, which is most common in shape files. Although when Google maps displays its maps, it uses SRID 3857 however when draw vectors on it it still expects the vector data to be SRID to 4326. For a more explanation of SRID, consult [Daniel's article](http://daniel-azuma.com/articles/georails/part-4)

Import the shape file
---------------------
Now let's import the data from the shape file into our new created *countries* table. We will use the *shp2pgsql* tool, which is already included in Postgres App. I assume that you already add the bin path into your PATH environment variable, if not, you can add them in your ~/.bash_profile

```
export PATH=$PATH:/Applications/Postgres.app/Contents/Versions/9.3/bin
```
Now you can open a new terminal and run *shp2pgsql* to check its usage,

```
$ shp2pgsql
RELEASE: 2.1.1 (r12113)
USAGE: shp2pgsql [<options>] <shapefile> [[<schema>.]<table>]
OPTIONS:
  -s [<from>:]<srid> Set the SRID field. Defaults to 0.
      Optionally reprojects from given SRID (cannot be used with -D).
 (-d|a|c|p) These are mutually exclusive options:
     -d  Drops the table, then recreates it and populates
         it with current shape file data.
     -a  Appends shape file into current table, must be
         exactly the same table schema.
     -c  Creates a new table and populates it, this is the
         default if you do not specify any options.
     -p  Prepare mode, only creates the table.
  -g <geocolumn> Specify the name of the geometry/geography column
      (mostly useful in append mode).
  -D  Use postgresql dump format (defaults to SQL insert statements).
  -e  Execute each statement individually, do not use a transaction.
      Not compatible with -D.
  -G  Use geography type (requires lon/lat data or -s to reproject).
  -k  Keep postgresql identifiers case.
  -i  Use int4 type for all integer dbf fields.
  -I  Create a spatial index on the geocolumn.
  -S  Generate simple geometries instead of MULTI geometries.
  -t <dimensionality> Force geometry to be one of '2D', '3DZ', '3DM', or '4D'
  -w  Output WKT instead of WKB.  Note that this can result in
      coordinate drift.
  -W <encoding> Specify the character encoding of Shape's
      attribute column. (default: "UTF-8")
  -N <policy> NULL geometries handling policy (insert*,skip,abort).
  -n  Only import DBF file.
  -T <tablespace> Specify the tablespace for the new table.
      Note that indexes will still use the default tablespace unless the
      -X flag is also used.
  -X <tablespace> Specify the tablespace for the table's indexes.
      This applies to the primary key, and the spatial index if
      the -I flag is used.
  -?  Display this help screen.
```

Now let's create another migration to import the shapefile,

```
$ rails g migration import_countries_from_shp
```

And in the migration file we have following contents,

{% codeblock lang:ruby %}
class ImportCountriesFromShp < ActiveRecord::Migration
  def up
    from_shp_sql = `shp2pgsql -c -g geom -W LATIN1 -s 4326 #{Rails.root.join('db', 'shpfiles', 'GAB_adm0.shp')} countries_ref`

    Country.transaction do
      execute from_shp_sql

      execute <<-SQL
          insert into countries(name, iso_code, geom) 
            select name_engli, iso, geom from countries_ref
      SQL

      drop_table :countries_ref
    end

  end

  def down
    Country.delete_all
  end

end
{% endcodeblock %}

In the *up* method of the migration, we first run the *shp2pgsql* in shell command by using Backticks(`) (check [this nice blog](http://tech.natemurray.com/2007/03/ruby-shell-commands.html)). The output of shp2pgsql is the SQL scripts and it will be returned from the result of the shell command, and then we store it in variable *from_shp_sql*. We pass following parameters to the *shp2pgsql*:

* **-c** Creates a new table and populates it
* **-g geom** Specify the geometry column name as *geom*
* **-W LATIN1** Set the encoding to LATIN1
* **-s 4326** set the SRID to 4326, since this is the SRID of the shape file

And then at last we pass the path of the shape file and name the generated table name as *countries_ref*.

And then we run this SQL in a transation to create the *countries_ref* table and import the data. And then run the following SQL to copy the data from table *countries_ref* to *countries*.

{% codeblock lang:sql %}
insert into countries(name, iso_code, geom) 
    select name_engli, iso, geom from countries_ref
 {% endcodeblock %}

 In the *countries_ref* table we already named the geometry column as *geom*. The *name_engli* and *iso* are attributes from the shape file and when shp2pgsql generate the table, it will name the column name the same as attribute name. At last since we already done the import we call *drop_table *countries_ref* to delete the *countries_ref* table. 

 In the *down* method, we just delete all data in the *countries* table.

 Generate GeoJSON
 ----------------
 [GeoJSON](http://geojson.org/) is a format for encoding a variety of geographic data structures. Google maps already has native support for it. In this blog let's create a static geojson file and render it in google map. In later blogs we will generate geojson format from database directly.

 RGeo has a nice addon [rgeo-geojson](https://github.com/rgeo/rgeo-geojson) which provides GeoJSON encoding and decoding services. Let's add it in Gemfile:

 {% codeblock lang:ruby %}
 gem 'rgeo-geojson'
 {% endcodeblock %}

 After bundle install, we can open the rails console (I have installed pry-rails to use pry instead of irb), 

 ```
 pry(main)> gabon = Country.first
 pry(main)> 
 pry(main)> factory = RGeo::GeoJSON::EntityFactory.instance
=> #<RGeo::GeoJSON::EntityFactory:0x007ffe2c7265c8>
 pry(main)> feature = factory.feature gabon.geom
 pry(main)> hash = RGeo::GeoJSON.encode feature
 pry(main)> File.open('gabon.json', 'w') {|file| file.write hash.to_json}
 ```

 Firstly we get the gabon model by calling *Country.first* since there is only one row in *countries* table. For rgeo-geojson, it uses a factory to generate the feature, so we get a factory and then calls the *feature* method and passes *gabon.geom*. This *geom* property is a RGeo geometry instance, and it's wrapped in *feature*. Then we call the *RGeo::GeoJson.encode feature* to encode it to a hash object, then at last we write the content of this hash to a file named *gabon.json*.

 Now let's move this *gabon.json* to the public folder so the browser can access it directly.

 When we start the server and access http://localhost:3000/gabon.json we should be able to see the content of this file in browser.

 Render in google maps
 ---------------------
 Now let's render this GeoJSON file on google maps. I have created a MapsController for that, and in the view it will create a *google.maps.Map* instance and render it. I won't explain all and you can check the source code in app/views/maps/index.html.erb. (Yeah I know it's very bad behavior to embed javascript in erb files directly, let's refactor it later.)

 The most important part is to call *map.data.loadGeoJson to load GeoJSON data,

 {% codeblock lang:javascript %}
 var mapOptions = {
          zoom: 1,
          mapTypeControlOptions: {mapTypeIds:
               [google.maps.MapTypeId.ROADMAP]}
        };

var mapElement = document.getElementById('mapDiv');
map = new google.maps.Map(mapElement, mapOptions);
map.setMapTypeId(google.maps.MapTypeId.ROADMAP);

//Load GeoJSON
map.data.loadGeoJson("/gabon.json");
{% endcodeblock %}

Now when we access http://localhost:3000/maps we can see that the the gabon area is filled with black color, as following figure,

{% img /images/gabon.png %}

The repository of this project is at https://github.com/climber2002/schoolpro . The source code of each blog has its own tag so it's easy to get it. For this part the source code is at https://github.com/climber2002/schoolpro/tree/part1 .

In this part we load the GeoJSON in a static file. In next part we will see how to generate the GeoJSON dynamically.
