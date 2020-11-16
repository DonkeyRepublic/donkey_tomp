# Donkey Republic x TOMP

This is a document that describes how Donkey Republic will work with TOMP API.
Please note that it is still work in progress and significant parts of the solution
may change.

## General Information
Donkey TOMP API is containerized into particular donkey cities. Therefore when calling
`/api/public/tomp/donkey_copenhagen/...` endpoints then it contains information only
aboutn the system operation in copenhagen

## Process Identifiers
TOMP processes used in Donkey:

Planning:
* `SPECIFIC_LOCATION_BASED` - planning is done from particular station
* `ATOMIC_PLANNING_AND_BOOKING` - booking intent planning should be immediatelly followed by booking

Booking:
* `ATOMIC_BOOKING_SET_IN_USE` - the bike (asset) should be marked as in use the moment it is booked

## Operator Information

Most important endpoints from the point of view of making a booking are
`/operator/stations` that contains basic information about stations (name, coordinates etc)
and `/operator/available-assets` which contains info about how many bike are available at given station
of each type (station can have both regular bikes and e-bikes for example).

Moreover, TOMP specification will probably soon have a place to store information about number of
available parking spaces at each station. Not sure which endpoint will have that information yet so this is
to be announced.

## Planning

As all data needed in exploratory mode of planning is available in operator information endpoints then
we will support going straight to booking intended planning with proper station selected.

Examples:

```
// Simple scenario - bike for 1 user, station has only bikes
// REQUEST

POST .../plannings&booking-intent=true
{
  "from": {
    "stationId": "1551"
  },
  "nrOfTravelers": 1
}

// RESPONSE

201 Created
{
  "validUntil": "2020-11-16T15:46:49.133Z",
  "options": [
    {
      "id": "FU-lA9P4MRWn1F8QkO8EiQ",
      "legs": [
        {
          "id": "839423832jIFwe"
          "assetType": {
            "id": "bike",
            "assetClass": "BICYCLE",
            "assetSubClass": "bike"
          },
          "pricing": {
            "planId": "17",
            "name": "Bike Price",
            "description": "Prices for Bikes",
            "taxable": false,
            "fare": {
              "estimated": false,
              "parts": [
                {
                  "amount": 12.5,
                  "units": 15,
                  "scaleFrom": 0,
                  "scaleTo": 15,
                  "scaleType": "MINUTE",
                  "currencyCode": "DKK",
                  "type": "FLEX",
                  "unit_type": "MINUTE",
                  "vatRate": 25
                },
                {
                  "amount": 2.5,
                  "units": 15,
                  "scaleFrom": 15,
                  "scaleTo": 30,
                  "scaleType": "MINUTE",
                  "currencyCode": "DKK",
                  "type": "FLEX",
                  "unit_type": "MINUTE",
                  "vatRate": 25
                },
                {
                  "amount": 15.0,
                  "units": 30,
                  "scaleFrom": 30,
                  "scaleTo": 60,
                  "scaleType": "MINUTE",
                  "currencyCode": "DKK",
                  "type": "FLEX",
                  "unit_type": "MINUTE",
                  "vatRate": 25
                },
                ....
              ]
            }
          }
        }
      ]
    }
  ]
}
```

```
// Scenario: bike for 1 user, there are bikes and ebikes in the station
// REQUEST

POST .../plannings&booking-intent=true
{
  "from": {
    "stationId": "1551"
  },
  "nrOfTravelers": 1
}

// RESPONSE

201 Created
{
  "validUntil": "2020-11-16T15:46:49.133Z",
  "options": [
    {
      "id": "FU-lA9P4MRWn1F8QkO8EiQ",
      "legs": [
        {
          "id": "839423832jIFwe"
          "assetType": {
            "id": "bike",
            "assetClass": "BICYCLE",
            "assetSubClass": "bike"
          },
          "pricing": {...}
        }
      ]
    },
    {
      "id": "f24n430FA324FOOCC21"",
      "legs": [
        {
          "id": "239fwefJJOQPBGEAZZ23"
          "assetType": {
            "id": "ebike",
            "assetClass": "BICYCLE",
            "assetSubClass": "ebike"
          },
          "pricing": {...}
        }
      ]
    }
  ]
}
```

```
// Scenario: more than 1 traveler
// REQUEST

POST .../plannings&booking-intent=true
{
  "from": {
    "stationId": "1551"
  },
  "nrOfTravelers": 3
}

// RESPONSE

201 Created
{
  "validUntil": "2020-11-16T15:46:49.133Z",
  "options": [
    {
      "id": "FU-lA9P4MRWn1F8QkO8EiQ",
      "legs": [
        {
          "id": "839423832jIFwe"
          "assetType": {
            "id": "bike",
            "assetClass": "BICYCLE",
            "assetSubClass": "bike"
          },
          "pricing": {...}
        },
        {
          "id": "239fwefJJOQPBGEAZZ23"
          "assetType": {
            "id": "bike",
            "assetClass": "BICYCLE",
            "assetSubClass": "bike"
          },
          "pricing": {...}
        },
        {
          "id": "993cweiijoAAXX2312"
          "assetType": {
            "id": "bike",
            "assetClass": "BICYCLE",
            "assetSubClass": "bike"
          },
          "pricing": {...}
        }
      ]
    }
  ]
}
```




