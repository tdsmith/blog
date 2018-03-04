---
title: "On Vancouver business license data"
layout: post
---

Yesterday was [Open Data BC](https://www.opendatabc.ca/)'s
[Open Data Day Hackathon](https://www.opendatabc.ca/pages/vodday-2018-vancouver-open-data-day-hackathon),
which was a great chance to learn about civic data sets
and spend some hacking on them
with data nerds and subject matter experts from the city and province.

One interesting data set is the city's
[business license registry](http://data.vancouver.ca/datacatalogue/businessLicence.htm),
which contains data all the way back to 1997.

Vancouver residents are technically required to hold a business license
to pursue almost any sort of non-employment income.
That includes renting property.

I didn't quite find a story I wanted to tell with the data,
but one surprising trend I noticed was that
the number of licenses associated with single-family housing rentals
(teal line)
has been declining year-over-year.

![Housing business licenses by year since 1997](/img/housing_licenses_by_year_overall.png){: width="800px" title="Housing business licenses by year since 1997"}

The trend is repeated in
[almost every neighborhood](/img/housing_business_licenses.pdf)
except for very built-up neighborhoods
with very few single-family structures
like the CBD, West End, and Fairview.

Some explanations for this trend could include:

* Poor awareness of the responsibility to hold a permit:
  the number of licenses issued appears much smaller than the number of
  housing units, so maybe all we're measuring is decreasing compliance.
  (I think this is probably the most likely interpretation.)
* Densification or multiple-unit conversions reducing the number
  of single-family structures available to rent.
  I think this is a factor in some neighborhoods,
  but I wouldn't expect the effect to be so uniform across the city.
* Increased occupancy of single-family structures by owners, or
  an increasing number of empty structures, as the expected investment
  return on the property swamps the expected profit on a rental.

Some more reliable data sets about housing in Vancouver include
[city building permits](http://data.vancouver.ca/datacatalogue/issuedBuildingPermits.htm),
which is only available for CY 2017 forward
but promises to be a valuable resource,
and [CMHC's information portal](https://www03.cmhc-schl.gc.ca/hmiportal/en/#Profile/1/1/Canada),
best accessed through
[Jens von Bergmann's](https://twitter.com/vb_jens)
[R package](https://github.com/mountainmath/cmhc).

Do you have another explanation?
What kind of stories would you like to be able to tell
about how and where businesses open and close in the city?
