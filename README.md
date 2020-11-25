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
    "coordinates": {
      "lng": 12.333,
      "lat": 55.123,
    }
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
          "id": "839423832jIFwe",
          "from": {
            "stationId": "123",
            "coordinates": {
              "lng": 12.333,
              "lat": 55.123
            }
          },
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
    "stationId": "123",
    "coordinates": {
      "lng": 12.333,
      "lat": 55.123
    }
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
          "id": "839423832jIFwe",
          "from": {
            "stationId": "123",
            "coordinates": {
              "lng": 12.333,
              "lat": 55.123
            }
          },
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
          "from": {
            "stationId": "123",
            "coordinates": {
              "lng": 12.333,
              "lat": 55.123
            }
          },
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
    "stationId": "1551",
    "coordinates": {
      "lng": 12.333,
      "lat": 55.123
    }
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
          "from": {
            "stationId": "123",
            "coordinates": {
              "lng": 12.333,
              "lat": 55.123
            }
          },
          "assetType": {
            "id": "bike",
            "assetClass": "BICYCLE",
            "assetSubClass": "bike"
          },
          "pricing": {...}
        },
        {
          "id": "239fwefJJOQPBGEAZZ23"
          "from": {
            "stationId": "123",
            "coordinates": {
              "lng": 12.333,
              "lat": 55.123
            }
          },
          "assetType": {
            "id": "bike",
            "assetClass": "BICYCLE",
            "assetSubClass": "bike"
          },
          "pricing": {...}
        },
        {
          "id": "993cweiijoAAXX2312"
          "from": {
            "stationId": "123",
            "coordinates": {
              "lng": 12.333,
              "lat": 55.123
            }
          },
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

## Booking
### Creating a booking

Booking should happen immediatelly after the planning produced booking options. Otherwise there is a risk of
someone fetching all the bikes from particular station. At this point booking is in state PENDING and legs are in
state NOT_STARTED. We don't provide access codes to the bike yet.

```
// REQUEST
POST /bookings/
{
  "id": "FU-lA9P4MRWn1F8QkO8EiQ",
  "customer": {
    "id": "1421322", // id of the customer in MP service
    "firstName": "John",
    "lastName": "Smith",
    "email": "john.smith@example.com",
    "phone": [
      {
        "number": "+123123123",
      }
    ]
  }
}


//RESPONSE
201 Created
{
  "id": "FU-lA9P4MRWn1F8QkO8EiQ",
  "bookingState": "PENDING",
  "customer": {
    "id": "1421322",
    "firstName": "John",
    "lastName": "Smith",
    "email": "john.smith@example.com",
    "phone": [
      {
        "number": "+123123123",
      }
    ]
  },
  "legs": [
    {
      "id": "839423832jIFwe",
      "from": {
        "stationId": "123",
        "coordinates": {
          "lng": 12.333,
          "lat": 55.123
        }
      },
      "state": "NOT_STARTED",
      "assetType": {
        "id": "bike",
        "assetClass": "BICYCLE",
        "assetSubClass": "bike"
      },
      "asset": {
        "id": "bike-12331",
        "overridenProperties": {
          "name": "Speedy"
        }
      },
      "pricing": {.... },
    }
  ]
}
```

### COMMITING THE BOOKING
Again, this should be done right after the booking is created. WHen this happens the booking is marked
as STARTED and we set legs in state PAUSED - it's because our system doesn't have a notion of reserved
bike - the moment it is booked the rental is started. Moreover, from this point on we start supplying
access codes required to open the bike.


```
// REQUEST
POST /bookings/FU-lA9P4MRWn1F8QkO8EiQ/events
{
  operation: "CONFIRM"
}

// RESPONSE
201 Created
{
  "id": "FU-lA9P4MRWn1F8QkO8EiQ",
  "bookingState": "PENDING",
  "customer": {
    "id": "1421322",
    "firstName": "John",
    "lastName": "Smith",
    "email": "john.smith@example.com",
    "phone": [
      {
        "number": "+123123123",
      }
    ]
  },
  "legs": [
    {
      "id": "839423832jIFwe",
      "state": "PAUSED",
      "departureTime": "2020-11-18T20:34:00Z",
      "from": {
        "stationId": "123",
        "coordinates": {
          "lng": 12.333,
          "lat": 55.123
        }
      },
      "assetType": {
        "id": "bike",
        "assetClass": "BICYCLE",
        "assetSubClass": "bike"
      },
      "asset": {
        "id": "bike-12331",
        "overridenProperties": {
          "name": "Speedy"
        }
      },
      "assetAccessData": {
        "validFrom": "2020-11-18T20:34:00Z",
        "validTo": "2020-11-19T20:34:00Z",
        "tokenType": "ekey",
        "tokenData": {
          "deviceName": "axa:1231442221",
          "btAddress": "AB:12:32:21:44:ES",
          "ekey": "9fwe9ui0fjewoif98weu0foiew...."
        }
      },
      "pricing": {.... },
    }
  ]
}
```

## Trip Execution Module
To be defined... Will (at least) be used to:
- Unlock and lock ebikes
- Update the rental with various events like unlock and lock for pedal bikes
- End the rental


## Support Module
To be defined... 
Will be used to handle support cases. Alternatively a direct connection between TO an MP ticketing system can also be set up.

## Payment Module
To be defined... 
Used by TO to instruct MP what to charge. This includes ride price after rental but also fees.
