---
layout: post
title: "Integrate PostGIS and Google maps in Rails Part 2"
date: 2014-05-25 22:49:16 +0800
comments: true
categories: [postgis, google maps, rails]
---

Here is the Part 2 of *Integrate PostGIS and Google maps in Rails*, in this part we will render the provinces of Gabon on Google map. Unlike Part 1 the GeoJSON data is fetched from a static file, in this part we will see how to fetch data from PostGIS and encode them into GeoJSON directly. The result is like the following figure, the 9 provinces are rendered on the map, and when the user put his mouse over one province, that area will be highlighted, and also on the top right there is a label which shows the name of the province that is highlighted.

{% img /images/province_highlighted.png %}

Create the Province model
-------------------------
The first thing we need to do is to create a **Province** model. Like how we created the **Country** model in part 1, we create a **Province** model, and then import the data in shape file into our PostGIS database. The main part in the migration is as following,

{% codeblock lang:ruby %}
class ImportProvincesFromShp < ActiveRecord::Migration
  def up
    from_shp_sql = `shp2pgsql -c -g geom -W LATIN1 -s 4326 #{Rails.root.join('db', 'shpfiles', 'GAB_adm1.shp')} provinces_ref`

    Province.transaction do
      execute from_shp_sql

      execute <<-SQL
          insert into provinces(name, geom) 
            select name_1, geom from provinces_ref
      SQL

      drop_table :provinces_ref
    end
  end

  def down
    Province.delete_all
  end
end
{% endcodeblock %}

The province data is in shape file *GAB_adm1.shp*, and we fetch the province name and geometry from the shape file and import them into *provinces* table (In the shape file province name is stored as attribute *name_1*).

Encode the feature
------------------
Now we need to fetch data from database and encode them as GeoJSON. GeoJSON is a JSON format for encoding geographic data structures. The following example is stolen from [the GeoJSON website](http://geojson.org/). We can see that in addition to the geometry data, the JSON can also encode additional properties.

{% codeblock lang:json %}
{
  "type": "Feature",
  "geometry": {
    "type": "Point",
    "coordinates": [125.6, 10.1]
  },
  "properties": {
    "name": "Dinagat Islands"
  }
}
{% endcodeblock %}

We will mainly use the *rgeo-geojson* gem to do the encoding. In *rgeo-geojson* gem, it converts a geometry type to a feature and then encode the feature to a hash, and then the hash can be rendered as a JSON.

 {% codeblock lang:ruby %}
province = Province.first
factory = RGeo::GeoJSON::EntityFactory.instance
feature = factory.feature province.geom
hash = RGeo::GeoJSON.encode feature
puts hash.to_json
 {% endcodeblock %}


We create a Concern named **Featurable**, which will add a *featurable* class method into the class which includes it. So for example in **Province** class, we include the **Featurable** like following,

 {% codeblock lang:ruby %}
class Province < ActiveRecord::Base
  include Featurable

  featurable :geom, [:name]
end
{% endcodeblock %}

The featurable accepts two parameters, the first is mandatory, which is the name of the attribute which contains the geometry data, the second parameter is an optional array of attribute names, when encoding it as GeoJSON, those attributes will be encoded as attributes. Here we encode the *name* attribute as the property of the feature. Now the **Province** class will have a instance method named **to_feature**, which returns a RGeo::GeoJSON::Feature.

{% codeblock lang:ruby %}
province = Province.first
feature = province.to_feature
puts RGeo::GeoJSON.encode(feature).to_json
{% endcodeblock %}

The result will display as following, it includes the name as its property and also has an id.
{% codeblock lang:json %}
{
  "type":"Feature",
  "geometry":{
    "type":"MultiPolygon",
    "coordinates":[[[[9.499305999999933,0.10763800000000856],
        ......,[9.748417000000074,1.0649910000000773]]]]
    },
  "properties":{
    "name":"Estuaire"
  },
  "id": 1
}
{% endcodeblock %}

Now let's have a look at how **Featurable** is implemented.
{% codeblock lang:ruby %}
module Featurable
  extend ActiveSupport::Concern

  module ClassMethods
    def featurable geom_attr_name, property_names = []
      define_method :to_feature do 
        factory = RGeo::GeoJSON::EntityFactory.instance

        property_names = Array(property_names)
        properties = property_names.inject({}) do |hash, property_name| 
          hash[property_name.to_sym] = read_attribute(property_name)
          hash
        end
        factory.feature read_attribute(geom_attr_name), self.id, properties
      end
    end


    # turns a collection of models to a feature collection
    # All models in the collection should implement the to_feature method
    def to_feature_collection models
      factory = RGeo::GeoJSON::EntityFactory.instance    
      features = models.map(&:to_feature)
      factory.feature_collection features
    end
  end
end
{% endcodeblock %}

In the implementation of *featurable* class method, when this method is called, the class will call *define_method* to define an instance method called *to_feature* and from line 9 to 13 it will generates a hash named *properties* whose key is the property name and value is the property value. And then on line 14 it calls the *RGeo::GeoJSON::EntityFactory#feature* method to create the feature and return it.

On line 21 it defines another class method called *to_feature_collection*, this is to convert a collection of models to a feature collection, for example the following code shows how to encode all 9 provinces as a feature collection.

{% codeblock lang:ruby %}
provinces = Province.all
feature_collection = Province.to_feature_collection provinces

{% endcodeblock %}

Provinces Controller
--------------------
Now let's create a **ProvincesController** to render all 9 provinces as GeoJSON.

{% codeblock lang:ruby %}

class ProvincesController < ApplicationController

  def index
      @provinces = Province.all

      respond_to do |format|
        format.json do  
          feature_collection = Province.to_feature_collection @provinces
          render json: RGeo::GeoJSON.encode(feature_collection)
        end

        format.html
      end
  end

end

{% endcodeblock %}

We can see that the *index* method responds to two formats, in the json format it will call *Province.to_feature_collection* to create the feature collection, and then calls *RGeo::GeoJSON.encode(feature_collection)* to encode the feature as a hash, and last calls *render* to render the hash as a JSON string.

In the *views/provinces/index.html.erb*, the main part is as following,

{% codeblock lang:javascript %}
//Create an info box for displaying names
var infoBox = document.createElement('div');
infoBox.innerHTML = "<div id='info-box'></div>";
map.controls[google.maps.ControlPosition.RIGHT_TOP].push(infoBox);

map.data.addListener('mouseover', function(event) {
  map.data.revertStyle();
  $('#info-box').text(event.feature.getProperty('name'));
  map.data.overrideStyle(event.feature, { fillColor: 'red' });
});

map.data.addListener('mouseout', function(event){
  map.data.revertStyle();
});

map.data.setStyle(function(feature) {
  return { fillColor: 'green',
          strokeWeight: 1}
});

{% endcodeblock %}

From line 2 to line 4 it creates an info box at the RIGHT TOP to display province name. And from line 6 to line 10 it adds a *mouseover* event listener, so when the mouse is over the province region, it set the text of the info box to the *name* property of the feature. And on line 12 it defines a *mouseout* event listener so the style is reverted when the mouse is out of the province region.

So in this part we showed how to fetch geometry data from PostGIS database and render them on google map. The source code for this part is at [github](https://github.com/climber2002/schoolpro/tree/part2).

