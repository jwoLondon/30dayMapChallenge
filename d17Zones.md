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

# 30 Day Map Challenge, Day 17: Zones

_This document best viewed in [litvis](https://github.com/gicentre/litvis)_

## Initial Thoughts

I like the idea that showing a polar azimuthal projection allows pie charts to be created in Vega-Lite, even though it wouldn't normally be considered as having a polar coordinates / pie chart facility. Combining a pie chart with a map of Antarctica gives an interesting chart / map hybrid. A possible value to map would be [territorial claims on the continent](https://en.wikipedia.org/wiki/Territorial_claims_in_Antarctica).

## Data Preparation

1. Downlaoded [Global country file including Antarctica](https://github.com/johan/world.geo.json/blob/master/countries.geo.json).
2. In mapshaper, filter only Antartica and save as topojson:

```
filter 'FID == "ATA"'
o format=topojson drop-table antarctica.json
```

3. Territorial zones copied directly from the [Wikipedia page](https://en.wikipedia.org/wiki/Territorial_claims_in_Antarctica#Antarctic_territorial_claims) and stored inline in the Vega-Lite spec.

## Map Design

Use an azimuthalEquidistant projection rotated by 90 degrees to centre on south pole.

```elm {v}
antarctica : Spec
antarctica =
    let
        data =
            dataFromUrl "data/antarctica.json" [ topojsonFeature "antarctica" ]

        proj =
            projection [ prType azimuthalEquidistant, prRotate 0 90 0 ]

        antarcticaSpec =
            asSpec [ geoshape [ maFill "lightgrey" ] ]

        graticuleSpec =
            asSpec
                [ graticule [ grStep ( 15, 5 ), grExtent ( -180, -60 ) ( 180, -90 ) ]
                , geoshape [ maFilled False, maStrokeWidth 0.3 ]
                ]
    in
    toVegaLite [ width 400, height 400, data, proj, layer [ antarcticaSpec, graticuleSpec ] ]
```

We can create a a function to generate set of 'pie' slices radiating out from the south pole as geoJson features. The `north` value allows the pie slices to be of different lengths so that the overlapping territorial claims by Argentina, Chile and UK can be differentiated.

```elm {l}
toGeojson : List ( ( String, Float, Float ), Float ) -> Spec
toGeojson zones =
    let
        inc =
            3

        seq a b x xs north =
            if x >= b then
                b :: xs |> List.reverse |> List.map (\lng -> ( lng, -60 - north ))

            else if modBy inc (ceiling x) == 0 then
                seq a b (x + 1) ((ceiling >> toFloat) x :: xs) north

            else
                seq a b (x + 1) xs north

        segments a b north =
            seq a b (a + 1) [ a ] north ++ [ ( b, -60 - north ), ( b, -90 ), ( a, -90 ) ]

        geom ( ( label, a, b ), north ) =
            geometry (geoPolygon [ segments a b north ]) [ ( "country", str label ) ]
    in
    geoFeatureCollection (List.map geom zones)
```

This allows us to overlay the wedges representing territorial claims on the Antartica map:

```elm {l}
antarcticaZones : Spec
antarcticaZones =
    let
        cfg =
            configure
                << configuration (coView [ vicoStroke Nothing ])

        w =
            1000

        h =
            w

        geoData =
            dataFromUrl "data/antarctica.json" [ topojsonFeature "antarctica" ]

        penguinData =
            dataFromColumns []
                << dataColumn "img" (strs [ "data/emperor.png" ])

        terrData =
            dataFromJson
                (toGeojson
                    [ ( ( "Argentina", -74, -25 ), 2 )
                    , ( ( "Australia", 142.033, 160 ), 6 )
                    , ( ( "Australia", 44.633, 136.017 ), 4 )
                    , ( ( "Chile", -90, -53 ), 3 )
                    , ( ( "France", 136.017, 142.033 ), 5 )
                    , ( ( "New Zealand", -180, -150 ), 7 )
                    , ( ( "New Zealand", 160, 180 ), 7 )
                    , ( ( "Norway", -20, 44.633 ), 5 )
                    , ( ( "United Kingdom", -80, -20 ), 1 )
                    ]
                )
                [ jsonProperty "features" ]

        labelData =
            dataFromColumns []
                << dataColumn "country" (strs [ "Argentina", "Australia", "Australia", "Chile", "France", "New Zealand", "Norway", "United Kingdom" ])
                << dataColumn "longitude" (nums [ -43, 150, 90, -85, 139.5, -177, 17, -23 ])
                << dataColumn "latitude" (nums [ -70, -70, -70, -71, -69, -74, -74, -67 ])

        proj =
            projection [ prType azimuthalEquidistant, prRotate 297 90 0 ]

        antarcticaSpec =
            asSpec [ geoData, geoshape [ maFill "lightgrey" ] ]

        graticuleSpec =
            asSpec
                [ graticule [ grStep ( 15, 5 ), grExtent ( -180, -60 ) ( 180, -90 ) ]
                , geoshape [ maFilled False, maStrokeWidth 0.3, maStroke "white" ]
                ]

        encZone =
            encoding
                << color [ mName "properties.country", mMType Nominal, mLegend [] ]

        terrSpec =
            asSpec [ terrData, encZone [], geoshape [ maOpacity 0.2 ] ]

        encLabel =
            encoding
                << position Longitude [ pName "longitude", pQuant ]
                << position Latitude [ pName "latitude", pQuant ]
                << text [ tName "country", tNominal ]

        labelSpec =
            asSpec [ labelData [], encLabel [], textMark [ maFont "Cinzel", maFontSize (w / 50) ] ]

        imgEnc =
            encoding
                << position X [ pNum (w * 0.5), pQuant ]
                << position Y [ pNum (h * 0.9), pQuant ]
                << url [ hName "img", hNominal ]

        penguinSpec =
            asSpec [ penguinData [], imgEnc [], image [ maWidth (w / 8), maHeight (h / 8) ] ]
    in
    toVegaLite
        [ cfg []
        , title "Territorial Claims\nin Antarctica"
            [ tiAnchor anStart
            , tiFont "Cinzel"
            , tiFontSize (w / 26)
            , tiOffset (w / -9.5)
            ]
        , width w
        , height h
        , proj
        , layer [ antarcticaSpec, terrSpec, graticuleSpec, labelSpec, penguinSpec ]
        ]
```

![day 17](images/day17.jpg)
