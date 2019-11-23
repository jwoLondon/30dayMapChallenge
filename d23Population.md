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

# 30 Day Map Challenge, Day 23: Population

_This document best viewed in [litvis](https://github.com/gicentre/litvis)_

## Initial Thoughts

I have avoided choropleth mapping so far, but would like to include at least one. Using a Gastner population scaled cartogram would be one way of illustrating the need avoid mapping people-related phenomena by land area.

## Data Preparation

1. Gastner boundaries from https://github.com/deldersveld/topojson

2. Election results from https://en.wikipedia.org/wiki/List_of_United_States_presidential_election_results_by_state, converted into CSV file selecting 1972-2016 elections.

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
    dataFromUrl "data/usElectionResults.csv" []


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

```elm {v interactive}
gastner : Spec
gastner =
    let
        cartogramData =
            dataFromUrl "data/usGastner2014Population.json" [ topojsonFeature "carto" ]

        proj =
            projection [ prType equalEarth, prRotate 100 0 0 ]

        trans =
            transTidy
                << lookup "shortName" cartogramData "properties.iso_3166_2" (luAs "gastnerStates")

        enc =
            encoding
                << shape [ mName "gastnerStates", mMType GeoFeature ]
                << color [ mName "result", mNominal, mScale colours, mLegend [] ]

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
            dataFromUrl "data/usStates.json" [ topojsonFeature "states" ]

        trans =
            transTidy
                << lookup "fips" geoData "id" (luAs "states")

        proj =
            projection [ prType albersUsa ]

        enc =
            encoding
                << shape [ mName "states", mMType GeoFeature ]
                << color [ mName "result", mNominal, mScale colours, mLegend [] ]

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
                    (strs
                        [ "data/nixon1972.jpg"
                        , "data/carter1976.jpg"
                        , "data/reagan1980.jpg"
                        , "data/reagan1984.jpg"
                        , "data/bush1988.jpg"
                        , "data/clinton1992.jpg"
                        , "data/clinton1996.jpg"
                        , "data/bush2000.jpg"
                        , "data/bush2004.jpg"
                        , "data/obama2008.jpg"
                        , "data/obama2012.jpg"
                        , "data/trump2016.jpg"
                        ]
                    )

        enc =
            encoding
                << position X [ pNum imgW ]
                << position Y [ pNum (h / 2) ]
                << url [ hName "president", hNominal ]

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
