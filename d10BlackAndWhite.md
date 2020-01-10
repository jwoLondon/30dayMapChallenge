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

# 30 Day Map Challenge, Day 10: Black and White

_This document best viewed in [litvis](https://github.com/gicentre/litvis)_

## Initial Thoughts

Lots of choices here. [Cycling profile map](https://openaccess.city.ac.uk/id/eprint/12351/7/wood_visualization_2015Postprint.pdf)?

Something that contrasts day and night? Could map Steve Abraham's One Year Time Trial heatmap for daylight and nighttime hours.

## Data Preparation

Steve's GPS data of were downloaded daily from Strava during 2015. Using location and timestamp of each GPS location, each point was classified as either in daylight or nighttime. The collection of day and night GPS points were converted into two density grids at a resolution of 300m. The grid was resampled to 600m cells and all non-zero grid cells were saved as a point files `abrahamDayPoints.csv` and `avrahamNightPoints.csv`.

Location of generated files:

```elm {l}
path : String -> String
path file =
    "https://gicentre.github.io/data/30dayMapChallenge/" ++ file
```

## Map Design

Adjacent day and night maps in a two-tone style.

```elm {l}
densityMap : Bool -> Spec
densityMap showDay =
    let
        w =
            500

        h =
            w * 2.104

        cfg =
            configure
                << configuration (coView [ vicoStroke Nothing ])

        boundaryData =
            dataFromUrl (path "gbCoastlineOSGB.json") [ topojsonFeature "coastline" ]

        proj =
            projection [ prType identityProjection, prReflectY True ]

        coastSpec =
            asSpec [ boundaryData, proj, geoshape [ maFilled False, maStrokeWidth 0.3, maStroke "#999" ] ]

        densityData =
            if showDay then
                dataFromUrl (path "abrahamDayPoints.csv") []

            else
                dataFromUrl (path "abrahamNightPoints.csv") []

        bgColour =
            if showDay then
                "white"

            else
                "black"

        fgColour =
            if showDay then
                "black"

            else
                "white"

        titleText =
            if showDay then
                "Day"

            else
                "Night"

        subtitleText =
            if showDay then
                """In 2015 Steve Abraham attempted to break the
               The record was set in 1939 by Tommy Godwin at

        Having to ride over 200 miles every day for a
        in daylight and"""

            else
                """world record for the furthest cycled in a year.
            75,065 miles and hadn't been beaten in 75 years.

      year, this is where Steve spent his time riding
      nightime hours."""

        titleAnchor =
            if showDay then
                anEnd

            else
                anStart

        colourRange =
            if showDay then
                [ 0, 2 ]

            else
                [ 1, -1 ]

        encPos =
            encoding
                << position X
                    [ pName "easting"
                    , pQuant
                    , pScale [ scZero False, scNice niFalse, scDomain (doNums [ 63820, 655620 ]) ]
                    , pAxis []
                    ]
                << position Y
                    [ pName "northing"
                    , pQuant
                    , pScale [ scZero False, scNice niFalse, scDomain (doNums [ -5000, 1240000 ]) ]
                    , pAxis []
                    ]

        encGlow =
            encPos
                << color
                    [ mName "density"
                    , mQuant
                    , mScale [ scScheme "greys" colourRange ]
                    , mLegend []
                    ]

        dSpec1 =
            asSpec [ encGlow [], circle [ maSize 20, maOpacity 0.1 ] ]

        dSpec2 =
            asSpec [ encPos [], circle [ maFill fgColour, maSize 1, maOpacity 1 ] ]

        densitySpec =
            asSpec [ densityData, layer [ dSpec1, dSpec2 ] ]
    in
    toVegaLite
        [ cfg []
        , width w
        , height h
        , title titleText
            [ tiSubtitle subtitleText
            , tiSubtitleColor fgColour
            , tiSubtitleFont "Roboto"
            , tiAnchor titleAnchor
            , tiColor fgColour
            , tiFontSize 60
            , tiOffset -40
            , tiFont "Roboto"
            ]
        , background bgColour
        , layer [ coastSpec, densitySpec ]
        ]
```

```elm{l v}
dayMap : Spec
dayMap =
    densityMap True


nightMap : Spec
nightMap =
    densityMap False
```

![day 10](images/day10.jpg)
