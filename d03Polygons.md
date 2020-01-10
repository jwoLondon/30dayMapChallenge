---
id: litvis

elm:
  source-directories:
    - /Users/jwo/common/elm/elm-vegalite/src/
---

@import "css/litvis.less"

```elm {l=hidden}
import VegaLite exposing (..)
```

# 30 Day Map Challenge, Day 3: Polygons

_This document best viewed in [litvis](https://github.com/gicentre/litvis)_

## Initial Thoughts

'Polygons' suggest geometry and geometry suggests Euclid. I am particularly taken to the design of Oliver Byrne's [visual treatment of Euclid's Elements](https://www.math.ubc.ca/~cass/euclid/byrne.html).

Possible ideas that might be inspired by Byrne's design:

- Building polygons from OSM?
- 'Squares' (from Trafalgar Square to garden squares to street names featuring 'square')?
- [Largest squares around the world](https://en.wikipedia.org/wiki/List_of_city_squares_by_size)?

## Data Preparation

### Buildings

1. [Open Street Map](https://www.openstreetmap.org/search?query=london#map=11/51.5077/-0.1274) with bounds `-0.1378,51.5106,-0.1122,51.5261` (central London)
2. Exported from OSM via Overpass API
3. Polygon features converted to geoJSON via `ogr2ogr -f GeoJSON bloomsburyPolys.geojson map.osm multipolygons`
4. Imported into [Mapshaper](https://mapshaper.org) and clipped in mapshaper console with `clip bbox=-0.1378,51.5106,-0.1122,51.5261`
5. Simplify geometry with `simplify 50%`
6. Extract buildings: `filter 'building != undefined'`
7. Store only OSM object ID: `filter-fields 'osm_way_id,osm_id'`
8. Store as topojson file: `o format=topojson bloomsburyBuildings.json`

### Footpaths

1. [Open Street Map](https://www.openstreetmap.org/search?query=london#map=11/51.5077/-0.1274) with bounds `-0.1378,51.5106,-0.1122,51.5261` (central London)
2. Exported from OSM via Overpass API
3. Line features converted to geoJSON via `ogr2ogr -f GeoJSON bloomsburyLines.geojson map.osm lines`
4. Imported into [Mapshaper](https://mapshaper.org) and clipped in mapshaper console with `clip bbox=-0.1378,51.5106,-0.1122,51.5261`
5. Extract footpaths: `filter 'highway == "footway"'`
6. Extreme simplify of path geometry with `simplify 2%`
7. Store as topojson file: `o format=topojson drop-table bloomsburyFootways.json`

Location of generated files:

```elm {l}
path : String -> String
path file =
    "https://gicentre.github.io/data/30dayMapChallenge/" ++ file
```

## Map Design

Buildings use the restricted four-colour palette of Byrne. Colours randomly assigned to the modulus 4 of the OSM id. Dashed lines to represent footways, chosen as these help to emphasise squares (Bloomsbury Square, Bedford Square, Lincoln's Inn Fields, Russell Square, Tavistock Square etc.)

```elm {l}
squaresMap : Spec
squaresMap =
    let
        cfg =
            configure
                << configuration (coView [ vicoStroke Nothing ])

        buildingData =
            dataFromUrl (path "bloomsburyBuildings.json")
                [ topojsonFeature "centralLondonPolys" ]

        footwayData =
            dataFromUrl (path "bloomsburyFootways.json")
                [ topojsonFeature "centralLondonLines" ]

        trans =
            transform
                << calculateAs "(isValid(datum.properties.osm_id)? datum.properties.osm_id:0 + isValid(datum.properties.osm_way_id)? datum.properties.osm_way_id:0)%4" "id"

        colours =
            categoricalDomainMap
                [ ( "0", "rgb(233,72,55)" )
                , ( "1", "rgb(57,58,104)" )
                , ( "2", "rgb(243,178,65)" )
                , ( "3", "rgb(25,15,17)" )
                ]

        buildingEnc =
            encoding
                << color [ mName "id", mNominal, mScale colours, mLegend [] ]

        buildingSpec =
            asSpec
                [ buildingData
                , trans []
                , buildingEnc []
                , geoshape [ maOpacity 0.9 ]
                ]

        footwaySpec =
            asSpec
                [ footwayData
                , geoshape
                    [ maOpacity 0.9
                    , maFilled False
                    , maStroke "black"
                    , maStrokeWidth 2
                    , maStrokeDash [ 4, 4 ]
                    ]
                ]
    in
    toVegaLite [ cfg [], width 1300, height 1300, layer [ footwaySpec, buildingSpec ] ]
```

![day 3](images/day03.jpg)
