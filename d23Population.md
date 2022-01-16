---
id: litvis

elm:
  dependencies:
    gicentre/elm-vegalite: latest
---

@import "css/litvis.less"

```elm {l=hidden}
import VegaLite exposing (..)
```

# 30 Day Map Challenge, Day 23: Population

_This document best viewed in [litvis](https://github.com/gicentre/litvis)_

## Initial Thoughts

I have avoided choropleth mapping so far, but would like to include at least one. Using a Gastner population scaled cartogram would be one way of illustrating the need avoid mapping people-related phenomena by land area.

## Data Preparation

1.  Gastner boundaries from [github.com/deldersveld/topojson](https://github.com/deldersveld/topojson)

2.  [Election results](https://en.wikipedia.org/wiki/List_of_United_States_presidential_election_results_by_state), converted into CSV file selecting 1972-2016 elections.

Location of generated files:

```elm {l}
path : String -> String
path file =
    "https://gicentre.github.io/data/30dayMapChallenge/" ++ file
```

## Map Design

Specifications common to both sets of maps.

```elm {l}
w =
    300


h =
    180


cfg =
    configure
        << configuration (coView [ vicoStroke Nothing ])
        << configuration (coHeader [ hdLabelFontSize 0, hdTitleFontSize 0 ])
        << configuration (coFacet [ facoSpacing 10 ])


electionData =
    dataFromUrl (path "usElectionResults.csv") []


colours =
    categoricalDomainMap
        [ ( "R", "rgb(220,72,55)" )
        , ( "D", "rgb(75,116,227)" )
        ]


transTidy =
    transform
        << foldAs [ "1972", "1976", "1980", "1984", "1988", "1992", "1996", "2000", "2004", "2008", "2012", "2016" ] "year" "result"
```

### Gastner Cartograms

```elm {l v interactive}
gastner : Spec
gastner =
    let
        cartogramData =
            dataFromUrl (path "usGastner2014Population.json") [ topojsonFeature "carto" ]

        proj =
            projection [ prType equalEarth, prRotate 100 0 0 ]

        trans =
            transTidy
                << lookup "shortName" cartogramData "properties.iso_3166_2" (luAs "gastnerStates")

        enc =
            encoding
                << shape [ mName "gastnerStates", mGeo ]
                << color [ mName "result", mScale colours, mLegend [] ]

        spec =
            asSpec
                [ width w
                , height h
                , trans []
                , proj
                , enc []
                , geoshape [ maStroke "black", maStrokeWidth 0.3, maStrokeOpacity 0.5 ]
                ]
    in
    toVegaLite
        [ cfg []
        , columns 1
        , electionData
        , facetFlow
            [ fName "year"
            , fOrdinal
            ]
        , specification spec
        ]
```

### Conventional Choropleths

```elm {v}
conventional : Spec
conventional =
    let
        geoData =
            dataFromUrl (path "usStates.json") [ topojsonFeature "states" ]

        trans =
            transTidy
                << lookup "fips" geoData "id" (luAs "states")

        proj =
            projection [ prType albersUsa ]

        enc =
            encoding
                << shape [ mName "states", mGeo ]
                << color [ mName "result", mScale colours, mLegend [] ]

        spec =
            asSpec
                [ width w
                , height h
                , proj
                , enc []
                , geoshape [ maStroke "black", maStrokeWidth 0.3, maStrokeOpacity 0.5 ]
                ]
    in
    toVegaLite
        [ cfg []
        , columns 1
        , electionData
        , trans []
        , facetFlow
            [ fName "year"
            , fOrdinal
            , fHeader [ hdLabelOrient siBottom, hdLabelPadding -20, hdLabelAlign haLeft ]
            ]
        , specification spec
        ]
```

### Presidents

```elm {v}
presidents : Spec
presidents =
    let
        imgW =
            100

        data =
            dataFromColumns []
                << dataColumn "year" (nums [ 1972, 1976, 1980, 1984, 1988, 1992, 1996, 2000, 2004, 2008, 2012, 2016 ])
                << dataColumn "president"
                    (List.map path
                        [ "nixon1972.jpg"
                        , "carter1976.jpg"
                        , "reagan1980.jpg"
                        , "reagan1984.jpg"
                        , "bush1988.jpg"
                        , "clinton1992.jpg"
                        , "clinton1996.jpg"
                        , "bush2000.jpg"
                        , "bush2004.jpg"
                        , "obama2008.jpg"
                        , "obama2012.jpg"
                        , "trump2016.jpg"
                        ]
                        |> strs
                    )

        enc =
            encoding
                << position X [ pNum imgW ]
                << position Y [ pNum (h / 2) ]
                << url [ hName "president" ]

        spec =
            asSpec [ width imgW, height h, enc [], image [ maHeight h ] ]
    in
    toVegaLite
        [ cfg []
        , data []
        , columns 1
        , facetFlow
            [ fName "year"
            , fOrdinal
            , fHeader
                [ hdLabelFont "Cinzel"
                , hdLabelFontSize 14
                , hdLabelOrient siBottom
                , hdLabelPadding -15
                ]
            ]
        , specification spec
        ]
```

![day 23](images/day23.jpg)
