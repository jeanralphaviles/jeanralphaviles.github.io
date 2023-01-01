---
layout: post
author: Jean-Ralph Aviles
title: "Reverse Engineering the National Weather Service Meteogram API"
description: "Reverse engineering the undocumented National Weather Service Meteogram API"
categories: reverse_engineering
tags: [reverse engineering weather.gov meteogram api national weather service nws]
date: 2022-03-16T23:23:00-6
---

[![alt text](/assets/pictures/meteogram.png "National Weather Service Meteogram.")]({% post_url 2022-03-16-weather-gov-meteograms %})

# Introduction

The National Weather Service [weather.gov](https://weather.gov) website is a
great resource for weather forecasts in the United States. It provides detailed
weather information with a simple, ad-free, non-bloated experience.

In particular, I have taken a liking to the NWS' meteogram format. Meteograms
are a graphical representation of weather forecasts with respect to time. The
NWS format is quick and easy to read.

The National Weather Service provides
[a well documented API](https://www.weather.gov/documentation/services-web-api) for
weather forecast information. Unfortunately, they do not provide an endpoint for
creating these meteograms. So, I spent an evening reverse engineering and
documenting the API. Enjoy!

# The National Weather Service Meteogram API

National Weather Service Meteograms are PNG files served at
<https://forecast.weather.gov/meteograms/Plotter.php>. There are multiple query
parameters required to generate an image.

Each query parameter is documented below. Advanced parameters are described in
greater detail.

| key    | description                                                                                                | default/example                                             |
|--------|------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| lat    | Forecast location latitude                                                                                 | 40.6521                                                     |
| lon    | Forecast location longitude                                                                                | -111.5067                                                   |
| wfo    | [NWS Forecast Office Identifier](https://www.weather.gov/srh/nwsoffices)                                   | SLC                                                         |
| zcode  | [NWS Public Forecast Zone Identifier](https://www.weather.gov/gis/PublicZones)                             | UTZ108                                                      |
| gset   | Unknown use. Must be an integer >= 1.                                                                      | 30                                                          |
| gdiff  | Unknown use. Must be an integer >= 0.                                                                      | 10                                                          |
| unit   | Display temperature/precipitation units in metric if units=1, otherwise imperial. Optional.                | 0                                                           |
| tinfo  | Timezone. Format {V,E,C,M,P,A,H}Y{Negative UTC Offset}. First letter is US timezone.                       | MY7                                                         |
| ahour  | Hours in the future to forecast.                                                                           | 0                                                           |
| pcmd   | 59 bits controlling which graphs to display.                                                               | 11011111111110000000000000000000000000000000000000000000000 |
| lg     | Language. English (en) or Spanish (sp). Optional.                                                          | en                                                          |
| indu   | Up to four integers controlling units of Surface Wind, Trans Wind, 20ft Wind, and Mixing Height. Optional. | 1!1!1                                                       |
| dd     | Display meteogram with dashes/dots if dd=1. Optional.                                                       | 0                                                           |
| bw     | Display meteogram in black and white if bw=1. Optional.                                                    | 0                                                           |
| hrspan | Hours to display. 6 <= hrspan <= 48. Optional.                                                             | 48                                                          |
| pqpfhr | Unknown use. Optional.                                                                                     | 6                                                           |
| psnwhr | Unknown use. Optional.                                                                                     | 6                                                           |

## wfo

This is the three letter
[NWS Forecast Office Identifier](https://www.weather.gov/srh/nwsoffices) for
the forecast area. For a given (lat, long) pair, you can find the identifier
with the <https://api.weather.gov/points> endpoint.

```bash
$ curl --silent 'https://api.weather.gov/points/40.6521,-111.5067' \
    | jq -r '.properties.cwa'
SLC
```

## zcode

This is the
[NWS Public Forecast Zone Identifier](https://www.weather.gov/gis/PublicZones)
for a location. It can also be determined through the
<https://api.weather.gov/points> endpoint.

```bash
$ curl --silent 'https://api.weather.gov/points/40.6521,-111.5067' \
    | jq -r '.properties.forecastZone[39:]'
UTZ108
```

## tinfo

This is the timezone to display data in. It consists of a timezone identifier,
a `Y`, and a UTC offset.

I'm pretty sure the valid timezone identifiers represent the following US
time zones.

| Identifier | Timezone |
|------------|----------|
| V          | Atlantic |
| E          | Eastern  |
| C          | Central  |
| M          | Mountain |
| P          | Pacific  |
| A          | Alaskan  |
| H          | Hawaiian |

For example `MY7` is Mountain Time with an offset of -7 hours. I believe the UTC
offset is set to one hour further back than the current timezone to cover the
current hour in the meteogram.

The timezone identifier seems to have no effect on the generated graph. It needs
to be set to one of the letters described however.

It is not possible to set a positive UTC offset.

## pcmd

The pcmd query parameter is 59 bits controlling which graphs should be included
in the generated PNG file. Each bit is described below.

| Bit | Graph                               |
|-----|-------------------------------------|
|   0 | Temperature (°F)                    |
|   1 | Dewpoint (°F)                       |
|   2 | Heat Index (°F)                     |
|   3 | Wind Chill (°F)                     |
|   4 | Surface Wind                        |
|   5 | Sky Cover (%)                       |
|   6 | Precipitation Potential (%)         |
|   7 | Relative Humidity (%)               |
|   8 | Rain                                |
|   9 | Thunder                             |
|  10 | Snow                                |
|  11 | Freezing Rain                       |
|  12 | Sleet                               |
|  13 | Freezing Spray                      |
|  14 | Fog                                 |
|  15 | Ceiling Height (x100ft)             |
|  16 | Visibility (mi)                     |
|  17 | Significant Wave Height (ft)        |
|  18 | Wave Period (sec)                   |
|  19 | Empty Graph                         |
|  20 | Mixing Height (x100ft)              |
|  21 | Haines Index                        |
|  22 | Lightning Activity Level            |
|  23 | Transport Wind (mph)                |
|  24 | 20ft Wind (mph)                     |
|  25 | Ventilation Rate (x1000 mph-ft)     |
|  26 | Swell Height (ft)                   |
|  27 | Swell Period (sec)                  |
|  28 | Swell 2 Height (ft)                 |
|  29 | Swell 2 Period (sec)                |
|  30 | Wind Wave Height (ft)               |
|  31 | Dispersion Index                    |
|  32 | Pressure (in)                       |
|  33 | Prob Wind 15mph                     |
|  34 | Prob Wind 25mph                     |
|  35 | Prob Wind 35mph                     |
|  36 | Prob Wind 45mph                     |
|  37 | Prob Wind Gust 20mph                |
|  38 | Prob Wind Gust 30mph                |
|  39 | Prob Wind Gust 40mph                |
|  40 | Prob Wind Gust 50mph                |
|  41 | Prob Wind Gust 60mph                |
|  42 | 6hr Prob QPF 0.1                    |
|  43 | 6hr Prob QPF 0.25                   |
|  44 | 6hr Prob QPF 0.5                    |
|  45 | 6hr Prob QPF 1.00                   |
|  46 | 6hr Prob QPF 2.00                   |
|  47 | 6hr Prob Snow 0.1in                 |
|  48 | 6hr Prob Snow 1in                   |
|  49 | 6hr Prob Snow 3in                   |
|  50 | 6hr Prob Snow 6in                   |
|  51 | 6hr Prob Snow 12in                  |
|  52 | Grassland Fire Danger Index         |
|  53 | Thunder Potential                   |
|  54 | Davis Stability Index               |
|  55 | Atmospheric Dispersion Index        |
|  56 | Low Visibility Ocurrence Risk Index |
|  57 | Turner Stability Index              |
|  58 | Red Flag Threat Index               |

## indu

This parameter consists of up to four values setting the units of the
following graphs. Each value is separated by an `!`.

| Bit | Graph             | 0  | 1   | 2             | 3             |
|-----|-------------------|----|-----|---------------|---------------|
| 0   | Surface Wind      | kt | mph | km/h          | m/s           |
| 1   | Transport Wind    | kt | mph | km/h          | m/s           |
| 2   | 20ft Wind         | kt | mph | km/h          | m/s           |
| 3   | Mixing Height     | ft | m   | no label (ft) | no label (ft) |

For example, `0!1!2!1`:

* Sets `Surface Wind` to `kt`
* Sets `Transport Wind` to `mph`
* Sets `20ft Wind` to `km/h`
* Sets `Mixing Height` to `m`

# Example

Putting all the query parameters together, we can generate the following
live Meteogram[1] for
[Park City, Utah](https://en.wikipedia.org/wiki/Park_City,_Utah).

[![](https://forecast.weather.gov/meteograms/Plotter.php?lat=40.6521&lon=-111.5067&wfo=SLC&zcode=UTZ108&gset=30&gdiff=10&unit=0&tinfo=MY7&ahour=0&pcmd=11011111111110000000000000000000000000000000000000000000000&lg=en&indu=1!1!1&dd=&bw=&hrspan=48&pqpfhr=6&psnwhr=6)](https://forecast.weather.gov/meteograms/Plotter.php?lat=40.6521&lon=-111.5067&wfo=SLC&zcode=UTZ108&gset=30&gdiff=10&unit=0&tinfo=MY7&ahour=0&pcmd=11011111111110000000000000000000000000000000000000000000000&lg=en&indu=1!1!1&dd=&bw=&hrspan=48&pqpfhr=6&psnwhr=6)

[1]:
<https://forecast.weather.gov/meteograms/Plotter.php?lat=40.6521&lon=-111.5067&wfo=SLC&zcode=UTZ108&gset=30&gdiff=10&unit=0&tinfo=MY7&ahour=0&pcmd=11011111111110000000000000000000000000000000000000000000000&lg=en&indu=1!1!1&dd=&bw=&hrspan=48&pqpfhr=6&psnwhr=6>
