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

# 30 Day Map Challenge, Day 15: Names

_This document best viewed in [litvis](https://github.com/gicentre/litvis)_

## Initial Thoughts

Would be a good opportunity to revisit our [BallotMap work](https://openaccess.city.ac.uk/436/) where we considered the role that candidate's location in London and position on the ballot paper influences their likelihood of being elected. Can use the faceted gridmap pattern from day 8 (green).

## Data Preparation

1.  Voting data were taken from the London GLA elections in 2010, stored in the file `londonVoting2010.csv`. For details of the data collection and cleaning process, see [Wood et al, 2011](https://openaccess.city.ac.uk/436/).

2.  Gridmap layout was adapted from the [After the Flood grid](https://aftertheflood.com/journal/we-created-a-new-data-service-to-benefit-citizens-for-future-cities-catapult/) but with `City of London` removed and some readjustment of boroughs to the east and south. Borough names and gridmap positions store in `londonBoroughCentroidsNoCity.csv`.

Location of generated files:

```elm {l}
path : String -> String
path file =
    "https://gicentre.github.io/data/30dayMapChallenge/" ++ file
```

## Map Design

Let's start with a non geographic representation that orders vertically the alpha position of each candidate (i.e. whether they were listed vertically first, second or third in their party on the ballot paper) and colours each candidate by their party rank (i.e. whether they were fist, second or third in their part in terms of number of votes received). This is similar to Figure 4 in [Wood, 2011](https://openaccess.city.ac.uk/436/) except that we are not sizing party by the proportion of votes.

```elm {l}
partyColours =
    categoricalDomainMap
        [ ( "Lab", "rgb(212,96,91)" )
        , ( "Lib Dem", "rgb(220,170,90)" )
        , ( "Cons", "rgb(86,120,165)" )
        ]
```

```elm {l v}
singleBoroughChart : Spec
singleBoroughChart =
    let
        cfg =
            configure
                << configuration (coView [ vicoStroke Nothing ])

        gSize =
            200

        ballotData =
            dataFromUrl (path "londonVoting2010.csv") []

        trans =
            transform
                << filter (fiExpr "datum.boroughGSS === 'E09000032'")
                << calculateAs "{'Lab': '1', 'Lib Dem': '2', 'Cons': '3'}[datum.party]" "sortedParty"
                << calculateAs "datum.sortedParty+datum.easting" "partyWard"

        colours =
            categoricalDomainMap
                [ ( "Lab", "rgb(212,96,91)" )
                , ( "Lib Dem", "rgb(220,170,90)" )
                , ( "Cons", "rgb(86,120,165)" )
                ]

        enc =
            encoding
                << position X [ pName "partyWard", pOrdinal, pSort [ soAscending ], pAxis [] ]
                << color [ mName "party", mScale partyColours ]
                << opacity [ mName "voteOrder", mOrdinal, mSort [ soDescending ], mScale [ scRange (raNums [ 0.15, 1 ]) ] ]
    in
    toVegaLite
        [ cfg []
        , spacing -1
        , ballotData
        , trans []
        , facet [ rowBy [ fName "ballotOrder", fOrdinal, fHeader [ hdLabelAngle 0 ] ] ]
        , specification (asSpec [ width gSize, enc [], bar [ maWidth 3.4, maHeight (gSize / 3) ] ])
        ]
```

Note that if there was no name bias in voting behaviour, there would be opaque and transparent bars spread equally among the three rows. But we see a systematic pattern of more first place candidates who are also alphabetically first within their party.

Faceting seems the obvious choice to arrange candidates into the three rows (name is alphabetically first in their party, second in their party, third in their party), but this will present problems when faceting the entire set of borough ballot charts.

An alternative is to explicitly position the coloured lines:

```elm {l v}
singleBoroughChart2 : Spec
singleBoroughChart2 =
    let
        cfg =
            configure
                << configuration (coView [ vicoStroke Nothing ])

        gSize =
            200

        ballotData =
            dataFromUrl (path "londonVoting2010.csv") []

        trans =
            transform
                << filter (fiExpr "datum.boroughGSS === 'E09000032'")
                << calculateAs "{'Lab': '1', 'Lib Dem': '2', 'Cons': '3'}[datum.party]" "sortedParty"
                << calculateAs "datum.sortedParty+datum.easting" "partyWard"
                << calculateAs "datum.ballotOrder - 1" "orderBaseline"

        enc =
            encoding
                << position X [ pName "partyWard", pOrdinal, pSort [ soAscending ], pAxis [] ]
                << position Y [ pName "ballotOrder", pOrdinal, pAxis [] ]
                << position Y2 [ pName "orderBaseline" ]
                << color [ mName "party", mScale partyColours ]
                << opacity [ mName "voteOrder", mOrdinal, mSort [ soDescending ], mScale [ scRange (raNums [ 0.15, 1 ]) ] ]
    in
    toVegaLite [ cfg [], width gSize, height gSize, ballotData, trans [], enc [], rect [] ]
```

We can facet the full dataset by the borough position to produce a ballot map, arranging each borough according to its geographic position.

```elm {l v}
ballotMap : Spec
ballotMap =
    let
        w =
            150

        h =
            w * 2 / 3

        cfg =
            configure
                << configuration (coView [ vicoStroke Nothing ])
                << configuration (coHeader [ hdLabelFontSize 0, hdTitleFontSize 0, hdTitlePadding -w ])

        legendProps =
            [ leTitleFont "Roboto Condensed"
            , leTitleFontWeight fwNormal
            , leTitleFontSize (w / 11.5)
            , leLabelFont "Roboto Condensed"
            , leLabelFontSize (w / 11.5)
            , leOrient loBottom
            , leOffset (-w / 1.9)
            , leSymbolStrokeWidth (w / 10)
            , leColumnPadding (w / -20)
            , lePadding (w / 25)
            ]

        boroughData =
            dataFromUrl (path "londonBoroughCentroidsNoCity.csv") []

        ballotData =
            dataFromUrl (path "londonVoting2010.csv") []

        trans =
            transform
                << lookup "boroughGSS" boroughData "GSSCode" (luFields [ "gridCol", "gridRow", "fullName" ])
                << calculateAs "{'Lab': '1', 'Lib Dem': '2', 'Cons': '3'}[datum.party]" "sortedParty"
                << window [ ( [ wiOp woRank ], "partyWard" ) ]
                    [ wiSort [ wiAscending "sortedParty", wiAscending "easting" ], wiGroupBy [ "boroughGSS" ] ]
                << calculateAs "datum.ballotOrder - 1" "orderBaseline"

        encBallot =
            encoding
                << position X [ pName "partyWard", pQuant, pAxis [] ]
                << position Y [ pName "ballotOrder", pQuant, pSort [ soDescending ], pAxis [] ]
                << position Y2 [ pName "orderBaseline" ]
                << color
                    [ mName "party"
                    , mScale partyColours
                    , mLegend
                        (leTitle "Each coloured line is a candidate\nordered vertically according to\ntheir position on the ballot paper."
                            :: legendProps
                        )
                    ]
                << opacity
                    [ mName "voteOrder"
                    , mOrdinal
                    , mSort [ soDescending ]
                    , mScale [ scRange (raNums [ 0.15, 1 ]) ]
                    , mLegend
                        (leTitle "Transparency indicates the rank of\na candidate within their party. Most\nvotes, rank 1, least votes rank 3."
                            :: legendProps
                        )
                    ]

        ruleSpec =
            asSpec [ encBallot [], rule [ maStrokeWidth (w * 3 / 200) ] ]

        transLabel =
            -- Needed so that we only show a borough label once for each set of voting data.
            transform
                << filter (fiExpr "datum.partyWard == 1 && datum.ballotOrder == 1")
                << calculateAs "split(datum.fullName,' ',1)" "mediumName"

        encLabel =
            encoding
                << text [ tName "mediumName" ]

        labelSpec =
            asSpec
                [ transLabel []
                , encLabel []
                , textMark [ maColor "#333", maX 0, maAlign haLeft, maDy (w / 2.2), maFontSize (w * 14 / 100), maFont "Roboto Condensed" ]
                ]

        ballotSpec =
            asSpec [ width w, height h, layer [ ruleSpec, labelSpec ] ]
    in
    toVegaLite
        [ cfg []
        , title "What's in a name?"
            [ tiFont "Fjalla One"
            , tiFontSize (w / 2.5)
            , tiOffset (w * -0.95)
            , tiSubtitle "London local election results showing relationship between\nalphabetical position on ballot paper and electorial success.\nCandidates higher up the ballot paper receive more votes."
            , tiSubtitleFont "Roboto Condensed"
            , tiSubtitleFontSize (w / 8)
            , tiSubtitleColor "#333"
            ]
        , ballotData
        , trans []
        , spacingRC (h / 5) (w / -5)
        , facet [ rowBy [ fName "gridRow", fQuant ], columnBy [ fName "gridCol", fQuant ] ]
        , specification ballotSpec
        ]
```

![day 15](images/day15.png)
