---
layout: post
title:  "Spatial querying of NetCDF files using MongoDB"
date:   2014-11-03 16:21:47
categories: jekyll update
---

The Problem
-----------

When you have a hammer, everything looks like a nail. These days, my toolbox is crammed full with schemaless databases... and today's nail is spatial querying of netCDF files. Let the NetCDF files are designed to be data dumpsters - anything can go in. For the most part, the data that goes into these files is model output, or remote sensing data. Typically this data is a set of values stored in the form of a 3-dimensional array with dimensions **[Latitude, Longitude, and Time]**. But there's nothing to prevent these dimensions from being **[Lions, Tigers, and Bears]**. Or the dimensions to be 42. It's a dumpster, folks. 

In practice, this means that there are no standard ways (that the author is aware of) to do geospatial queries directly on NetCDF files. So something like:

{% highlight python %}
my_location = Princeton
my_data = my_netcdf_file.find(location=Princeton)
{% endhighlight %}

Is just not going to happen.

Indeed, good luck finding **any** standard file format of **any** type that allows geospatial queries. Seriously, please find me one; I'd love to make this tutorial obsolete. 

Bottom line: The data dumpsters don't care about your geocoding.   

Space Madness
=============

More to the point, even if NetCDF *was* spatially-aware, it wouldn't be very graceful about finding the nearest grid cell to an arbitrary location. The reason is twofold:

1. Space is curved. Especially the surface of the planet. So nearest points aren't simply euclidian distances between carteisan coordinates (Sorry, Mrs. Teely!)

2. Finding things in space is hard. The issue here is doing a fast query. It's easy to see that 'aaab' and 'aaaa' are close to each other as strings, but it's not as easy to see that -29.34,39.56 is close to -30.12,40.89... The share no common values, so any matching requires knowledge of the coordinate system in which these points exist. This is - to use a classic English phrase - ['Difficult, Difficult, Lemon, Difficult'][lemon]


The Solution
------------

Geospatial Databases
====================

In absence of a sane geospatial indexing system within NetCDF, we need a lightweight and speedy way to find locations and lookup data. We need a geospatial database. Or, let's just say, we need a database. But which one? Most databases [aren't any better][geospatial] at handling geospatial stuff than NetCDF itself. [MongoDB][mongodb] is a document-based database system, with documents stored as [JSON][json] objects. [GeoJSON][geojson] is a system for defining geographic entities within JSON documents. For example, a JSON object containing a single coordinate location might look like: 

{% highlight json %}
{
  "geometry": {
    "type": "Point",
    "coordinates": [125.6, 10.1]
  },
}
{% endhighlight %}

That's nice and readable and it's pretty clear what's going on. Just remember that the Longitude is always first in any coordinate pair (because [X,Y], right?) and you're all set. The keywords ``type`` and ``coordinates`` are essential. And of course this document could have any other content you might wish to include:

{% highlight json %}
{
  "geometry": {
    "type": "Point",
    "coordinates": [125.6, 97.1]
  },
  "feature": {
    "type":"Island",
    "name":"Misfit Toys",
    "inhabitant":"Charlie-in-a-Box"
  },
  "status": "fictional"
}
{% endhighlight %}


Ok. So we can store locations in a database. Good on us. However, referring back to the issues defined above, we see that the issue is geospatial **queries**, not geospatial storage. It's always about the queries. So how does MongoDB handle spatial queries?

Making a Hash of it
===================

As discussed earlier, it's hard to deal with spatial data because similar points don't look similar by any simple comparison. In the limit, it's easy to understand this. ``Princeton, NJ`` is close to ``Kingston, NJ``, but how does a database sort that out? Even with the coordinate locations, it's hard to infer proximity - you can only know things are nearby if you understand the coordinate system itself! And that's just determining the distance between *two* points. If you want to determine what town in New Jersey a particlar location is closest to, you need to solving a system of quadratic distance equations between the selected point and all other locations within the database. The problem now scales with the database size, so you're hosed. 


MongoDB solves this problem by using [geohashes][geohash] to store (and index) geospatial locations. These hashes transform a coorindate location into a binary string, where each bit corresponds to a division in coordinate space. For example, the first bit specifying longitude sets wether the point is in the range 0-180 for ``1``, or -180-0 for ``0``. The second bit would further specify the division within the prior range, so ``10`` would specify that the point is in the range 0-90 longitude. The more bits provided, the more accurate the specification is. Furthermore, nearby locations will have many similar bits, which can be compared very, very, rapidly (bitwise calculations are the fundamental unit of computing). So the longitude 42.6 can be written as ``101111001001`` (prove this to yourself sometime while waiting for the bus). In reality, the geohashes are written such that lon/lat alternate within a single string. And there are some work-arounds for dealing with edge and corner cases (literally). [Read all about it][geohash_info]


The Plan
--------

1. Create a MongoDB database that holds all of the locations contained within a NetCDF file. This amounts to creating one entry per grid cell, where each entry contains the lat/lon of the center of the grid cell.

2. Index this database on the location field so that it can be queried for spatial proximity. 

3. Given a random location ``A``, query the database to determine the grid cell ``B`` whose center is nearest to the location ``A`` . 

4. Query the NetCDF file to return all of the data it has for grid cell ``B``.

5. Profit.


The Code
--------

Step 0. Setup 
=============

You need to have [brew][brew] installed on your machine. It helps if you know how to use [virtualenv][virtualenv]. 'What the hell is virtualenv', you say? [Go here][virtualenv_explained]. It helps to also install [virtualenvwrapper][virtualenvwrapper], which makes managing virtual environments much easier. Create a new virtual environment, activate it, and ``cd`` into it. If you're using virtualenvwrapper, this can all be done in one step with ``mkproject <whatever>``.

We need to install mongodb, so ``brew install mongodb``. Once that's done, open up a terminal and start the mongodb server using the ``mongod`` command. Leave that running.

> Tip: you'll need to always be sure mongod is started before trying to 
> read or write to a local database. Don't worry, python will yell 
> at you if you forget.

We also need a lot of libraries. Hopefully, you already have these. If not, do:

{% highlight bash %}
> brew tap python
> brew install numpy
> brew install netcdf
> brew tap homebrew/science 
> brew install hdf5
{% endhighlight %}

Do a ``brew doctor`` after all this and clean up whatever messes you've made. You may need to do a few ``brew link <package>`` runs. I just overwrite the old files, and let brew handle this. Others may have a smarter idea. Let's move on.

Once you've set up a virtual environment, you need to gather up some libraries.
First you need to ``pip install pymongo``. Then, because we're going to be using netcdf files, you need to grab some other stuff. 

{% highlight bash %}
> pip install 
> pip install numpy
> pip install netcdf4
{% endhighlight %}


Step 1. Creating a MongoDB database.
====================================

We could create the database within a mongodb terminal, but we're going to use Python's [pymongo][pymongo] library, which we installed into our virtualenv in Step 0. 

{% highlight python %}
from pymongo import MongoClient

client = MongoClient()      # defaults to using localhost 
db = client.geolocations    # create a new database called 'geolocations'
netcdf = db.netcdf          # create a new collection called 'netcdf' 

# Create an initial dummy location, which we can remove later.
# This isn't really necessary, but we do it for illustrative purposes.
location = {
            "type":"test",
            "grid_cell":
                {
                    "type":"Point",
                    "coordinates": [125.6, 10.1]
                }
            }

netcdf.insert(location) # write that location to the database

{% endhighlight %}

At this point, we now have a single record in our database. If you run these commands at the python prompt, you will see the ``ObjectID`` of the record, which is created by mongodb when it writes a new record (e.g. ``ObjectId('545930ef4322dd7d13b8431c')``). These are UUIDs, and you don't really need to worry about them.


Step 2. Creating a geospatial index in MongoDB.
===============================================

To create the index, we will need to:

* Open the NetCDF file we are interested in.

* Read every coordinate location from the file (this is crazy tedious, but we only have to do it once!)

* Create a new GeoJSON object with the coordinate location

* Write the location to the database. [create_database][create_database]

{% highlight python %}
from netCDF4 import Dataset
from pymongo import MongoClient

# Open our geolocations database and the netcdf collection:
client = MongoClient()      
db = client.geolocations   
netcdf = db.netcdf         

# Open the NetCDF file we are interested in.
this_data = Dataset('prec_19480101.nc', format='NETCDF4')

# What's in this thing? 
# We should see latitude and longitude, or some such...
print this_data.variables 

# Our dataset contains 'lat' and 'lon':
lat = this_data.variables['lat']
lon = this_data.variables['lon']

# Read every coordinate location from the file 
for i in range(len(lon)):
    locations = []
    for j in range(len(lat)):
        location = {
            "grid_cell": {
                "type":"Point",
                "coordinates":[lon[i],lat[j]]
            },
            # Add the x and y locations to this record:
            "x":i, 
            "y":j,
            "dataset":"Africa Drought Monitor"
        }
        locations.append(location)
    # Write locations to the database
    netcdf.insert(locations)

{% endhighlight %}

But we can do better. Let's build this in the cloud and host the database at compose.io. Once we've setup an account, and create a deployment called 'geolocations'. That creates a cloud mongodb server that we can put our data into. There is a URL that you can use to connect. Find it in mongolab somewhere.

Here's ours:
``mongodb://<dbuser>:<dbpassword>@ds051640.mongolab.com:51640/geolocations``. You will need to add your username and password for the db (not the same as mongolab user and password!)



{% highlight python %}
from netCDF4 import Dataset
from pymongo import MongoClient
import os

# You need to set these environmental variables with the user info for 
# the database you are using on MongoLab.
MONGOLAB_USER = os.getenv('MONGOLAB_USER')
MONGOLAB_PASSWORD = os.getenv('MONGOLAB_PASSWORD')

mongolab_url = 'mongodb://%s:%s@ds051640.mongolab.com:51640/geolocations' % \
    (MONGOLAB_USER, MONGOLAB_PASSWORD)

# Open our geolocations database and the netcdf collection:
client = MongoClient(mongolab_url)      
db = client.geolocations   
netcdf = db.netcdf         

# Open the NetCDF file we are interested in.
this_data = Dataset('prec_19480101.nc', format='NETCDF4')

# What's in this thing? 
# We should see latitude and longitude, or some such...
print this_data.variables 

# Our dataset contains 'lat' and 'lon':
lat = this_data.variables['lat']
lon = this_data.variables['lon']

# Read every coordinate location from the file 
for i in range(len(lon)):
    locations = []
    for j in range(len(lat)):
        location = {
            "grid_cell": {
                "type":"Point",
                "coordinates":[lon[i],lat[j]]
            },
            # Add the x and y locations to this record:
            "x":i, 
            "y":j,
            "dataset":"Africa Drought Monitor"
        }
        locations.append(location)
    # Write locations to the database
    netcdf.insert(locations)

{% endhighlight %}


Step 3. Querying the MongoDB database.
======================================

To query the database, we use pymongo, just the same as we used to build the database.


{% highlight python %}
from netCDF4 import Dataset
from pymongo import MongoClient
import os

# You need to set these environmental variables with the user info for 
# the database you are using on MongoLab.
MONGOLAB_USER = os.getenv('MONGOLAB_USER')
MONGOLAB_PASSWORD = os.getenv('MONGOLAB_PASSWORD')

mongolab_url = 'mongodb://%s:%s@ds051640.mongolab.com:51640/geolocations' % \
    (MONGOLAB_USER, MONGOLAB_PASSWORD)

# Open our geolocations database and the netcdf collection:
client = MongoClient(mongolab_url)      
geolocations_db = client.geolocations   
netcdf_collection = geolocations_db.netcdf         

# Open the NetCDF file we are interested in.
this_data = Dataset('prec_20070216.nc', format='NETCDF4')

# We're interested in Band1:
variable='Band1'

# So, let's do a query (we're looking for Lusaka today)
my_lat = -15.4167
my_lon = 28.2833

# Let's find the nearest grid cell center to Lusaka:
my_grid_cell = netcdf_collection.find({
    'grid_cell':{
        '$near':{
            '$geometry':{'type':"Point", 'coordinates': [my_lon,my_lat]}
        }
    }
}).limit(1)[0]

# Here's the point in the netCDF file that we want:
my_x = my_grid_cell['x']
my_y = my_grid_cell['y']

# Here's the data we want:
this_data.variables[variable][0][my_x][my_y]

{% endhighlight %}


Step 4. Retrieving data from the NetCDF file.
=============================================

It's easy to get at the data we want, as long as we know the variable name 
and the structure of the netCDF file.
Watch out for dummy dimensions.

{% highlight python %}

# Here's the point in the netCDF file that we want:
my_x = my_grid_cell['x']
my_y = my_grid_cell['y']

# Here's the data we want:
this_data.variables[variable][0][my_x][my_y]

{% endhighlight %}

Virtual Lookups
---------------

Now let's move the whole thing to the cloud. To do this, we take the code we just wrote, wrap in a simple web server and put it on the cloud. 

Then this:

{% highlight python %}

# Let's find the nearest grid cell center to Lusaka:
my_grid_cell = netcdf_collection.find({
    'grid_cell':{
        '$near':{
            '$geometry':{'type':"Point", 'coordinates': [my_lon,my_lat]}
        }
    }
}).limit(1)[0]

{% endhighlight %}

becomes: 
``my_grid_cell = requests.get(url,data={'coordinates': [my_lon,my_lat]})``


Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
[lemon]: https://www.youtube.com/watch?v=7mAFiPVs3tM
[geojson]: http://geojson.org
[mongodb]: http://www.mongodb.org
[json]: http://www.json.org
[geospatial]: http://ralphbarbagallo.com/2011/04/02/an-overview-of-geospatial-databases/
[geohash]: http://en.wikipedia.org/wiki/Geohash
[geohash_info]: http://www.bigfastblog.com/geohash-intro
[virtualenvwrapper]: http://virtualenvwrapper.readthedocs.org
[pymongo]: http://api.mongodb.org/python/current/tutorial.html
[brew]: http://brew.sh
[virtualenv]: https://virtualenv.pypa.io/en/latest/
[virtualenv_explained]: http://blog.saevon.ca/coding/python-virtual-environments-and-pip/
[create_database]: https://gist.github.com/kcaylor/b208f89796b03b86271c




