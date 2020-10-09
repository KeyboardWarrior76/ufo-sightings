# UFO Hotspots 2.0 Directions

## Overview
Thanks for taking the time to look at my work! For this project I tried to use a react frontend (I never got to build it out), rails api backend, and a postgis (postgresql with gis native extension) database. 

First I made the requested commands for populating sightings from our CSV file and outputing the json file with sightings by hotspot. The sightings:populate command takes about a minute depending on your hardware, and the sightings:json_by_hotspot command takes about ten seconds. I used the Prallel gem to try and speed the execution of these.

It also has a controller for performing CRUD actions on sightings (a short reference is provided below). There are few concerns to manage the logic for filtering, pagination, and location in models/concerns.

I should also explain why I chose to use PostGIS over Mysql for this challenge. PostGIS has the most robust native features for location related queries and its own data types relating to that. Mysql can certainly do location queries as well, but it's not optimized in the same way. PostGIS should be faster (all else equal).

My Database schema (just pasted from db/schema.rb) looks like:
```
create_table "hotspots", force: :cascade do |t|
  t.string "name"
  t.geography "lonlat", limit: {:srid=>4326, :type=>"st_point", :geographic=>true}, null: false
  t.datetime "created_at", precision: 6, null: false
  t.datetime "updated_at", precision: 6, null: false
  t.index ["lonlat"], name: "index_hotspots_on_lonlat", using: :gist
end

create_table "sightings", force: :cascade do |t|
  t.datetime "sighting_date"
  t.string "shape"
  t.string "city"
  t.string "state"
  t.integer "duration_seconds"
  t.string "duration_string"
  t.text "comments"
  t.geography "lonlat", limit: {:srid=>4326, :type=>"st_point", :geographic=>true}, null: false
  t.datetime "created_at", precision: 6, null: false
  t.datetime "updated_at", precision: 6, null: false
  t.index ["lonlat"], name: "index_sightings_on_lonlat", using: :gist
end
```

Hotspots have a name & lonlat. 
Sightings have: sighting_date, shape, city, state, duration_seconds, duration_string, comments, & lonlat.

The 'lonlat' attribute is of PostGIS's st_point data type with a default srid of 4326. This puts the point on a geographical coordinate system.
The lonlat attribute has also been indexed for performance.

I have to admit, I didn't really know what to do with the schemadump.sql file as it relates to a rails app. I thought it was a best practice for ruby/rails schemas should be generated through rails 
migrations (but tell me if I'm wrong about that).

## Feedback
I faced a lot of challenges in making this, the biggest of which was just setting up docker. In fact, I spent more time learning about docker than making the project! I've been needing to learn about containerization for another project I want to make, so I welcomed the excuse to get ahead of that. Beside the docker setup, I had good number of difficulties with the write permissions on rails generated files. I read that I can specify user permissions in docker, but with my time constraints I had to just keep going.

My next steps would have been to add rack attack (for rate limiting and throttling), build out the CRUD actions for hotspots, and then get to work on the frontend. I thought up a decent program I wanted to make, but I just ran out of time. I also need to refactor a few files. The rake tasks in particular need to be cleaned up.

This challenge was really fun overall. Especially being UFO/location related, it's nice to work with data that's not always in my day job. I think it would be nice to add some clarity on what is required vs what's up to your own discretion to make. The bullet point list of objectives trails off at the end and implies that we should be creative, but the commands in the beginning of the list are part of the main requirements. Granted, I could have asked more queestions and clarified certain things (which could also be a dimension of the test), but with my busy week it was difficult to invest the right amount of time.

Thanks again for your time!

## Resources
You can find the following resources at..

* docker-compose.yml - root directory
* Dockerfile - app/backend/Dockerfile & app/frontend/Dockerfile
* JSON output - app/backend/output/hotspots_with_sightings.json

## Setup Commands
* sudo docker-compose up
* sudo docker exec ufo-backend rails db:setup
* sudo docker exec ufo-backend rails sightings:populate
  --(takes about a minute)

A ufo-hotspots/sightings json file is already provided in the app/backend/output directory, but i can be re-generated by using the command:
sudo docker exec ufo-backend rails sightings:json_by_hotspot

## API Endpoint
The API is accessible at 127.0.0.1:3001. You can perform CRUD actions and the index action can be filtered and paginated; including a filter for all sightings within a given radius of a location.

### Example Create/POST Request:
```
curl -d 'sighting_date=1972-12-01 03:16:56 +0000' -d 'shape=Circle' -d 'duration=10' -d 'comments=foo' -d 'city=Salt Lake' -d 'state=Utah' -d 'latitude=39.23333' -d 'longitude=-98.334309' http://127.0.0.1:3001/api/sightings
```

### Example Update/PUT Request:
```
curl -X PUT -d 'sighting_date=1972-12-01 03:16:56 +0000' -d 'shape=Circular' -d 'duration=1000' -d 'comments=Nick was here. He saw everything' -d 'city=Salt Lake' -d 'state=Utah' -d 'latitude=39.23333' -d 'longitude=-98.334309' http://127.0.0.1:3001/api/sightings/80068
```

### Example Show/GET Request:
```
curl http://127.0.0.1:3001/api/sightings/80068
```

### Example Show/Index Requests:
Requests can be paginated and filtered by:
  * shape: string
  * city: string
  * state: string
  * duration_seconds: integer
  * radius: integer (miles)
  * latitude: float (required for radius)
  * longitude: float (required for radius)
  * limit: integer
  * page: integer
  * direction: [asc, desc]
  * order_by: [created_at, updated_at, duration_seconds, shape, sighting_date]

The index route also sends back the total count of records and the total number of pages

All sightings within one mile of Washington DC
```
curl -G -d 'radius=1' -d 'latitude=38.897663' -d 'longitude=-77.036575' http://127.0.0.1:3001/api/sightings
```

The first ten results from Wyoming with a disk shape
```
curl -G -d 'limit=10' -d 'shape=disk' -d 'state=wy' http://127.0.0.1:3001/api/sightings
```

The 10 most recent sightings
```
curl -G -d 'limit=10' -d 'order_by=sighting_date' -d 'direction=desc' http://127.0.0.1:3001/api/sightings
```