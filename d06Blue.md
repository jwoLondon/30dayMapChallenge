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

# 30 Day Map Challenge, Day 6: Blue

_This document best viewed in [litvis](https://github.com/gicentre/litvis)_

## Initial Thoughts

Blue line networks? Could be rivers or perhaps rectilinear patterns of drainage network in flat area (East Anglia, Somerset Levels?)

Or perhaps Wales, indicating mountainous areas?

## Data Preparation

1. Use [ONS boundary data](https://geoportal.statistics.gov.uk/datasets/european-electoral-regions-december-2016-full-clipped-boundaries-in-great-britain).

2. Select the Wales region, simplify, reproject to longitude/latitude WGS84 and save the Wales boundary polygon:

```
mapshaper gbRegions.json \
  -filter 'eer16nm == "Wales"' \
  -simplify 10% \
  -proj +init=EPSG:4326 \
  -rename-layers Wales \
  -o format=topojson wales.json
```

2. Download Ordnance Survey [OpenRivers dataset](https://www.ordnancesurvey.co.uk/opendatadownload/products.html) as shapefiles.

3. Project stream nodes from OS National Grid to longitude/latitude WGS84, clip to the Wales boundary and filter junction/source/outlet nodes:

```
mapshaper HydroNode.shp \
  -proj +init=EPSG:4326 \
  -clip wales.json \
  -filter 'formOfNode == "junction" || formOfNode == "source" || formOfNode == "outlet"' \
  -filter-fields 'formOfNode' \
  -each 'longitude=this.x.toFixed(3), latitude=this.y.toFixed(3)' -o 'walesStreamNodes.csv'
```

4. Project watercourse links from OS National Grid to longitude/latitude WGS84, clip to the Wales boundary and perform simplification:

```
mapshaper WatercourseLink.shp \
  -proj +init=EPSG:4326 \
  -clip wales.json \
  -simplify 1% \
  -o format=topojson drop-table walesBlueLines.json
```

Location of generated files:

```elm {l}
path : String -> String
path file =
    "https://gicentre.github.io/data/30dayMapChallenge/" ++ file
```

## Map Design

A conventional map design in shades of blue. Use layers to give subtle glow to line features and coastal boundary.

```elm {l v interactive}
streamTopologyMap : Spec
streamTopologyMap =
    let
        cfg =
            configure
                << configuration (coView [ vicoStroke Nothing ])

        walesData =
            dataFromUrl (path "wales.json") [ topojsonFeature "Wales" ]

        linkData =
            dataFromUrl (path "walesBlueLines.json") [ topojsonFeature "WatercourseLink" ]

        nodeData =
            dataFromUrl (path "walesStreamNodes.csv") []

        trans =
            transform
                << filter (fiExpr "datum.formOfNode == 'junction'")

        proj =
            projection [ prType transverseMercator ]

        lineSpec =
            asSpec [ linkData, geoshape [ maFilled False, maStroke "blue", maStrokeWidth 4, maOpacity 0.1, maStrokeCap caRound ] ]

        lineSpec2 =
            asSpec [ linkData, geoshape [ maFilled False, maStroke "blue", maStrokeWidth 0.8, maStrokeCap caRound ] ]

        walesSpec =
            asSpec [ walesData, geoshape [ maFilled False, maStroke "white", maStrokeJoin joRound, maStrokeWidth 30, maOpacity 0.5 ] ]

        walesSpec2 =
            asSpec [ walesData, geoshape [ maFill "rgb(225,225,255)", maStroke "blue", maStrokeWidth 0.6 ] ]

        nodeEnc =
            encoding
                << position Longitude [ pName "longitude", pQuant ]
                << position Latitude [ pName "latitude", pQuant ]

        nodeSpec =
            asSpec
                [ nodeData
                , nodeEnc []
                , trans []
                , point [ maStrokeWidth 0.4, maSize 5, maFilled True, maFill "white", maStroke "blue", maOpacity 0.9 ]
                ]
    in
    toVegaLite
        [ cfg []
        , title "                   The Rivers of Wales" [ tiFont "Cinzel", tiColor "rgb(0,26,200)", tiFontSize 78, tiOrient siBottom, tiOffset -190, tiAnchor anStart ]
        , width 2200
        , height 2400
        , background "rgb(238,238,255)"
        , proj
        , layer [ walesSpec, walesSpec2, lineSpec, lineSpec2, nodeSpec ]
        ]
```

![day 6](images/day06.jpg)
