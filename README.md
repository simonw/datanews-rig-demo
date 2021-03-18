# Data News Rig Demo

This is me building and documenting a data news rig to help community newsrooms make and publish data projects.

I'm trying to keep things as simple and minimal and replicable as possible, primarily using Github and AWS.

## Psuedo-plan

Build a basic map of US vaccination data. Possibilities include:

- Scrape US Vaccination data
- Build a US map using mapshaper
    - Classify colors based on vaccine data
    - Make State labels
    - Use just state innerlines
- Use Svelte to format the map, header, etc.
- Enhance that map:
    - CSS tooltips
    - Key
    - Make it responsive using CSS
- Deploy it to a production site
- Deploy it to a dev site
- Use a CSS/JS packager
- Have the map update every day ... automatically
- Use Pym.js to make it embeddable
- Can I get the data into Datasette?
- Can I drive a map off Datasette?
- Do I need a data-processing step in my deployment?

## Publishing on main

Right now, I've set it so anything in the `public` directory will be uploaded to `https://projects.datanews.studio/[repo-name]` when code is pushed to the the `main` branch.

I'm planning to change that so it's from the `prod` branch.

Note to self: The cache on that site is currently set to 30 mins. So I need to invalidate the Cloudfront cache when uploading new code!

Currently, that trick only works for **public** repositories, though I've got some ideas of how to make this work for private ones, which will be important for client work.

## Let's shape a map

### Install mapshaper

```
npm init -y
npm install mapshaper --save
```

### Get base map

Getting states map from the [US Census site](https://www.census.gov/geographies/mapping-files/time-series/geo/carto-boundary-file.html).

See my [map-making repo](https://github.com/ReallyGoodSmarts/map-making/README.md) for how I made the `us_states_albers.json` file using mapshaper.

## Get the vaccination data

The CDC data behind its [vaccination map](https://covid.cdc.gov/covid-data-tracker/#vaccinations) is in a [JSON file at this address](https://covid.cdc.gov/covid-data-tracker/COVIDData/getAjaxData?id=vaccination_data).

We can download it with `curl` and insure it's nicely formatted with `jq`:

```
curl https://covid.cdc.gov/covid-data-tracker/COVIDData/getAjaxData?id=vaccination_data | jq .> data/vaccinations.json
```

There's a little problem with this that's going to trip me up if we don't fix it. The vaccination data actually is in an array one level down in the json, under the key `vaccination_data`:

```
{
  "runid": 1857,
  "vaccination_data": [
    {
      "Date": "2021-03-16",
      "Location": "US",
      "ShortName": "USA",
      "LongName": "United States",
      "Census2019": 331996199,
      "date_type": "Report",
      "Doses_Distributed": 142918525,
      "Doses_Administered": 110737856,
      "Dist_Per_100K": 43048,
      "Admin_Per_100K": 33355,
      "Administered_Moderna": 53647312,
      "Administered_Pfizer": 55393141,
      ...
```

I didn't know about [jq](https://stedolan.github.io/jq/) before. It's a very cool way to handle json data. Looking ahead, I know we're going to need the data in CSV form to merge using mapshaper, and to do that we're going to need that at the top level. So I'm going to make it easy by downloading it that way by turning the `.` into `.vaccination_data`:

```
curl https://covid.cdc.gov/covid-data-tracker/COVIDData/getAjaxData?id=vaccination_data | jq .vaccination_data > data/vaccinations.json
```

## Turn the vax data into a CSV for mapshaper

Joining the data to the map turns out to be very easy with mapshaper, but we'll need the vaccination data as a CSV.

We can use jq for that as well, thanks a [blog post](https://www.freecodecamp.org/news/how-to-transform-json-to-csv-using-jq-in-the-command-line-4fa7939558bf/) about how to convert it and [a code playground](https://jqplay.org/s/QOs3d_fMLU) laying it out.

As explaind in a freeCodeCamp blog post, this is the key line of code:

```
jq -r '(map(keys) | add | unique) as $cols | map(. as $row | $cols | map($row[.])) as $rows | $cols, $rows[] | @csv'
```

The full conversion line is:

```
cat data/vaccinations.json | jq -r '(map(keys) | add | unique) as $cols | map(. as $row | $cols | map($row[.])) as $rows | $cols, $rows[] | @csv' > data/vaccinations.csv
```

## Join the data to the map

I'll use mapshaper's `-join` command:

```
npx mapshaper data/us_states_albers.json -join data/vaccinations.csv keys=STUSPS,Location \
    -o public/vaccinations_map.json 
```

## Classify the map

Mapshaper will calculate classifications for you, or you can define them yourself and mapshaper will assign colors for you. It's all with the `-classify` command. 

There are [many color ramps](https://github.com/d3/d3-scale-chromatic) you can use.

I actually _don't_ want mapshaper to quantize or otherwise slice up my data, because I'm really interested in which states are at which percentage of people fully innoculated. (As of this writing, it's still under 20% nationwide).

I've added commands to classify by colors, and I used `style` to color the strokes of the states. I also switched the output to `svg` by adding `.svg` to the end of the file name. The full command is now:

```
npx mapshaper data/us_states_albers.json -join data/vaccinations.csv keys=STUSPS,Location \
    -classify field=Administered_Dose2_Pop_Pct color-scheme=PuBuGn breaks=10,20,30,40,50,60,70,80,90 \
    -style stroke=#c2c2c2 stroke-width=1 \
    -o public/vaccinations_map.svg 
```

## Label the states

With mapshaper, labels get attached to points. But I have polygons (the states). But mapshaper also lets you make a new point layer based on the center points of a set of polygons. So I'm going to try to:

- Make a point layer from my existing polygon layer
- Add labels to those points
- Display them all together (may have to solve layer-ordering)

Read up on an [issue Matt Bloch answered](https://github.com/mbloch/mapshaper/issues/422), and his guide to [working with layers](https://github.com/mbloch/mapshaper/wiki/Introduction-to-the-Command-Line-Tool#working-with-layers), and pretty quickly built three scripts to make different label map layers using the additional commands:

```
-points inner \
-style label-text=NAME \
```

I've combined them all into one shell script to [make state label layers](https://github.com/ReallyGoodSmarts/map-making/blob/main/scripts/make_state_labels.sh), and also added `id`'s for each label (so I can move them with CSS later) and gave them all a class.

These live in my [`map-making` repo](https://github.com/ReallyGoodSmarts/map-making), under the `geos/` folder for future use and reference.

## Innerlines

One of the elegant things the New York Times does with its national maps is to "remove" the outer perimeter boundaries, leaving the shape of the US to be created by the colored polygons. In fact, what they've really done is made an "innerlines" layer for boundaries between states.

You can make innerlines with mapshaper, which very nicely calculates those lines based on boundaries two polygons share. I've [made US state innerlines](https://github.com/ReallyGoodSmarts/map-making/blob/main/scripts/make_state_innerlines.sh) and place them in my [`map-making` repo](https://github.com/ReallyGoodSmarts/map-making) for future use.

## Blend state shapes, innerlines and labels

Since I've made the innerlines and labels maps in my [`map-making` repo](https://github.com/ReallyGoodSmarts/map-making), I'm going to copy over those files (from the `geos` directory) into this project's `data` directory and layer them in.

My mapshaper command up to this point is:

```
npx mapshaper data/us_states_albers.json name=states \
    -join data/vaccinations.csv keys=STUSPS,Location \
    -classify field=Administered_Dose2_Pop_Pct color-scheme=PuBuGn breaks=10,20,30,40,50,60,70,80,90 \
    -style stroke=#c2c2c2 stroke-width=1 \
    -o public/vaccinations_map.svg 
```

But using information about [mapshaper's layers](https://github.com/mbloch/mapshaper/wiki/Introduction-to-the-Command-Line-Tool#working-with-layers) and a [stackoverflow post](https://gis.stackexchange.com/questions/206170/how-to-merge-combine-files-in-mapshaper-using-combine-files-command), I'm refactoring that to load in all the layers:

```
npx mapshaper -i data/us_states_albers.json data/us_states_albers_labels_nyt.json data/us_states_albers_innerlines.json combine-files \
    -rename-layers states,names,innerlines \
    -join data/vaccinations.csv keys=STUSPS,Location target=states\
    -classify field=Administered_Dose2_Pop_Pct color-scheme=PuBuGn breaks=10,20,30,40,50,60,70,80,90 target=states \
    -style stroke=#c2c2c2 stroke-width=1 target=names \
    -style stroke=#c2c2c2 stroke-width=1 target=innerlines \
    -o public/vaccinations_map.svg target=*
```

Key things to note:

- `combine-files` added to the input command, to bring them in as separate files (I was stumped by this for a while)
- `rename-layers` to give the layers proper names
- `target=` added to each line, to declare what layer to involve/target with the command









