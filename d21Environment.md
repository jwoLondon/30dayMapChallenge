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

# 30 Day Map Challenge, Day 21: Environment

_This document best viewed in [litvis](https://github.com/gicentre/litvis)_

## Initial Thoughts

Would like to map air pollution in some way. Could embed air [pollution charts](https://twitter.com/jwoLondon/status/1023616910829740033) in a London map, but probably too much detail would be lost. Also technically challenging with Vega-Lite if main London map is layered.

Could map the familiar London Tube Map but with line width proportional to PM2.5 pollution levels, following the [FT investigation](https://www.ft.com/content/6f381ad4-fef7-11e9-be59-e49b2a136b8d). Assisted by the fact that a relatively limited (albeit important) part of the network is measured.

Use isotype symbolisation to show premature deaths due to air pollutants. One way of turning intangible statistics into more meaningful representations. Could take the estimate for London [around 9000 according to the Kings College report](https://www.london.gov.uk/sites/default/files/hiainlondon_kingsreport_14072015_final.pdf) and distribute those 9000 deaths as 9000 person icons using a white ghost style.

## Data Preparation

Given the nature of long term exposure to NO2 and PM2.5, it is impossible to geolocate single locations for a premature death, So instead we can distribute the 9000 locations randomly within a set of locations associated with road traffic.

1. In mapshaper, dissolve London boroughs (from `londonBoroughs.json`) to form single outline polygon:

```
dissolve2
o format=topojson londonBoundary.json
```

2. Extract all point locations associated with London 'highways' in Open Street Map:

```
ogr2ogr -f GeoJSON m25Points.geojson map.osm points
mapshaper m25Points.geojson -clip londonBoundary.json \
 -filter 'highway != undefined && highway != "street_lamp"' \
 -filter-fields highway \
 -each 'longitude=this.x, latitude=this.y' -o 'londonHighwayPoints.csv'
```

Location of generated files:

```elm {l}
path : String -> String
path file =
    "https://gicentre.github.io/data/30dayMapChallenge/" ++ file
```

## Map Design

Following the rhetorical visualization design approach exemplified in [Risk, Cycling and Denominator Neglect](https://www.gicentre.net/blog/2013/11/24/risk-cycling-and-denominator-neglect), we can symbolise each modelled premature death with a person symbol and locate them somewhere on the map of London. By random sampling from the distribution of highway-related we can control the number of symbols displayed but spread them out over London closer to the road network. Displaying a full year of 9000 deaths is probably too many symbols to grasp visually, so more effective may be to split by month and repeat 12 times (adding to the rhetorical effect).

Additional minor monthly variation assuming slight increases in summer months reflecting increased low-level ozone pollution and 'summer smog' events.

```elm {l}
ghostMap : String -> Float -> Spec
ghostMap month n =
    let
        w =
            1000

        h =
            w * 0.8

        cfg =
            configure
                << configuration (coView [ vicoStroke Nothing ])

        locationData =
            dataFromUrl (path "londonHighwayPoints.csv") []

        boundaryData =
            dataFromUrl (path "londonBoundary.json") [ topojsonFeature "london" ]

        person =
            "M1.7 -1.7h-0.8c0.3 -0.2 0.6 -0.5 0.6 -0.9c0 -0.6 -0.4 -1 -1 -1c-0.6 0 -1 0.4 -1 1c0 0.4 0.2 0.7 0.6 0.9h-0.8c-0.4 0 -0.7 0.3 -0.7 0.6v1.9c0 0.3 0.3 0.6 0.6 0.6h0.2c0 0 0 0.1 0 0.1v1.9c0 0.3 0.2 0.6 0.3 0.6h1.3c0.2 0 0.3 -0.3 0.3 -0.6v-1.8c0 0 0 -0.1 0 -0.1h0.2c0.3 0 0.6 -0.3 0.6 -0.6v-2c0.2 -0.3 -0.1 -0.6 -0.4 -0.6z"

        trans =
            transform
                << sample n

        posEnc =
            encoding
                << position Longitude [ pName "longitude", pQuant ]
                << position Latitude [ pName "latitude", pQuant ]

        shapeEnc =
            posEnc
                << shape [ mPath person ]

        textEnc =
            posEnc
                << text [ tStr "= one premature death", tQuant ]

        boundarySpec =
            asSpec
                [ boundaryData
                , geoshape
                    [ maFilled True
                    , maStroke "white"
                    , maStrokeJoin joRound
                    , maStrokeWidth 3
                    , maOpacity 0.2
                    ]
                ]

        boundaryGlowSpec =
            asSpec
                [ boundaryData
                , geoshape
                    [ maFilled True
                    , maStrokeJoin joRound
                    , maStroke "white"
                    , maStrokeWidth 8
                    , maOpacity 0.15
                    ]
                ]

        personSpec =
            asSpec
                [ locationData
                , trans []
                , shapeEnc []
                , point
                    [ maSize 20
                    , maFilled True
                    , maColor "white"
                    , maOpacity 0.5
                    , maStroke "white"
                    , maStrokeWidth 2
                    , maStrokeOpacity 0.9
                    ]
                ]

        leData =
            dataFromColumns []
                << dataColumn "longitude" (nums [ -0.49 ])
                << dataColumn "latitude" (nums [ 51.29 ])

        leSymbolSpec =
            asSpec
                [ leData []
                , shapeEnc []
                , point
                    [ maSize 100
                    , maFilled True
                    , maColor "white"
                    , maOpacity 0.8
                    , maStroke "white"
                    ]
                ]

        leTextSpec =
            asSpec
                [ leData []
                , textEnc []
                , textMark
                    [ maColor "white"
                    , maFont "Raleway"
                    , maFontWeight Normal
                    , maFontSize 32
                    , maAlign haLeft
                    , maDx 25
                    ]
                ]
    in
    toVegaLite
        [ width w
        , height h
        , cfg []
        , title ("People who died in " ++ month ++ " because we polluted London's air")
            [ tiColor "white"
            , tiFont "Raleway"
            , tiAnchor anStart
            , tiFontWeight Normal
            , tiFontSize 32
            ]
        , background "black"
        , padding (paSize 40)
        , layer [ boundaryGlowSpec, boundarySpec, personSpec, leSymbolSpec, leTextSpec ]
        ]
```

^^^elm {v=(ghostMap "the last month" 695)}^^^
^^^elm {v=(ghostMap "October" 772)}^^^
^^^elm {v=(ghostMap "September" 799)}^^^
^^^elm {v=(ghostMap "August" 879)}^^^
^^^elm {v=(ghostMap "July" 925)}^^^
^^^elm {v=(ghostMap "June" 851)}^^^
^^^elm {v=(ghostMap "May" 826)}^^^
^^^elm {v=(ghostMap "April" 747)}^^^
^^^elm {v=(ghostMap "March" 719)}^^^
^^^elm {v=(ghostMap "February" 601)}^^^
^^^elm {v=(ghostMap "January" 535)}^^^
^^^elm {v=(ghostMap "December" 650)}^^^

![day 21](images/day21.jpg)
