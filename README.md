# Donkey Republic x TOMP

This is a document that describes how Donkey Republic will work with TOMP API.
Please note that it is still work in progress and significant parts of the solution
may change.

# Table of Contents
* [General Information](#general-information)
* [Process Identifiers](#process-identifiers)
* [How to use the API](#how-to-use-the-api)
  * [Api Key and other headers](#api-key-and-other-headers)
  * [Errors](#errors)
* [A word about pricing](#a-word-about-pricing)
* [Lifecycle of a rental made with TOMP](#lifecycle-of-a-rental-made-with-tomp)
  * [Creating a booking](#creating-a-booking)
  * [Cancelling booking](#cancelling-booing)
  * [Unlocking and locking bike](#unlocking-and-locking-bike)
  * [Ending rental](#ending-rental)
  * [Fetching final price](#fetching-final-price)
* [Endpoints](#endpoints)
  * [Cities](#cities)
  * [Operator](#operator)
    * [Information](#information)
    * [Stations](#stations)
    * [Available assets](#available-assets)
    * [Pricing plans](#pricing-plans)
  * [Plannings](#plannings)
    * [Planning create](#planning-create)
  * [Bookings](#bookings)
    * [Booking create](#booking-create)
    * [Get booking](#get-booking)
    * [Booking events](#booking-events)
  * [Legs](#legs)
    * [Fetch leg](#fetch-leg)
  * [Trip execution (Leg Events)](#trip-execution-leg-events)
    * [Locking and unlocking](#locking-and-unlocking)
    * [Finishing rental](#finishing-rental)
    * [Refresh ekey](#refresh-ekey)
  * [Payment](#payment)
    * [Journal entries](#journal-entries)
* [Webhooks](#webhooks)
  * [Leg Events Webhook](#leg-events-webhook)
  * [Additional costs](#additional-costs)
  * [Testing Webhooks](#testing-webhooks)

## General Information
### Donkey Republic in various cities
Donkey TOMP API is containerized into particular donkey cities. Therefore when calling
`/api/aggregators/tomp/donkey_rotterdam/...` endpoints then it contains information only
about the system operation in copenhagen.

## Process Identifiers
TOMP processes used in Donkey:

Planning:
* `SPECIFIC_LOCATION_BASED` - planning is done from particular station
* `ATOMIC_PLANNING_AND_BOOKING` - booking intent planning should be immediatelly followed by booking

## How to use the API
### Api key and other headers

All API Requests need to be authenticated with the API key, the body of POST requests should be in json format and the request should accept json format in response. Like so:

```
POST /api/aggregators/tomp/donkey_rotterdam/plannings?booking_intent=true  HTTP/1.1
Content-Type: application/json
Accept: application/json
X-Api-Key: TheApiKey
...other headers like host, content length etc

{
  "from": {
      "stationId": "3628"
  },
  "nrOfTravelers": 1
}
```

### Errors
There is a general guideline about how errors should be handled in TOMP api described [here](https://github.com/TOMP-WG/TOMP-API/wiki/Error-handling-in-TOMP)

The first indication about what kind of error you got is HTTP status code and then more information is represented in returned json that looks like this:

```
{
  "errorcode": 2002,
  "title": "Invalid parameters",
  "detail": "Invalid stationId"
}
```

There are 2 error types that don't guarantee proper json response and those are errors with following HTTP statuses:
* `500` - unexpected error
* `404` - not found

## A word about pricing

The pricing in our system is build in such a way to promote longer rentals so that people don't need
to feel rushed on a bike and can keep the bike longer even if they are taking pauses (going to the restaurant
or a shop).

The example pricing could look like the following:

![Pricing](images/pricing.jpeg)

Which basically means:
* If your ride is up to 15 minutes - the price is 1.5 EUR
* If your ride is between 15 and 30 minutes - the price is 2 EUR
* If between 30 minutes and 1 hour - 3 EUR

[TOMP's pricing definition](https://github.com/TOMP-WG/TOMP-API/wiki/Payment#in-depth-the-fare-object) doesn't directly support
such a pricing but it does have a scaling type of pricing that can handle an example given in TOMP's documentation:
> bike rental, 1.50USD per half hour for the first hour, after this 2.50USD per hour

We are leveraging this scaling model to produce our pricing. So the example Donkey pricing seen in the picture above would look like this:

```
[
  {
    "amount": 1.5,
    "units": 15,
    "scaleFrom": 0,
    "scaleTo": 15,
    "scaleType": "MINUTE",
    "currencyCode": "EUR",
    "type": "FLEX",
    "unitType": "MINUTE",
    "vatRate": 21.0
  },
  {
    "amount": 0.5,
    "units": 15,
    "scaleFrom": 15,
    "scaleTo": 30,
    "scaleType": "MINUTE",
    "currencyCode": "EUR",
    "type": "FLEX",
    "unitType": "MINUTE",
    "vatRate": 21.0
  },
  {
    "amount": 1.0,
    "units": 30,
    "scaleFrom": 30,
    "scaleTo": 60,
    "scaleType": "MINUTE",
    "currencyCode": "EUR",
    "type": "FLEX",
    "unitType": "MINUTE",
    "vatRate": 21.0
  },
  {
    "amount": 2.0,
    "units": 60,
    "scaleFrom": 60,
    "scaleTo": 120,
    "scaleType": "MINUTE",
    "currencyCode": "EUR",
    "type": "FLEX",
    "unitType": "MINUTE",
    "vatRate": 21.0
  },
  ...
]
```

So basically our pricing in TOMP's terms is defined as:
* For first 15 minutes of the ride we charge 1.5 EUR per 15 minutes
* Later up to 30 minutes of the ride we charge 0.5 EUR per 15 minutes
* Later up to 60 minutes of the ride we charge 1 EUR per 30 minutes
* Then up to 120 minutes of the ride we charge 2 EUR per 60 minutes

Our pricings also are usually defined with set amounts till 1 or 2 days of rental time.
Then we have a pricing item that says what's the price for each additional day.
In case of this example pricing we had prices defined up to 72 hours and then we had the additional
day price point which will look like that (notice that `scaletTo` is not defined which means
that this is the pricing for the rest of the ride):


```
  {
    "amount": 7.0,
    "units": 1440,
    "scaleFrom": 4320,
    "scaleType": "MINUTE",
    "currencyCode": "EUR",
    "type": "FLEX",
    "unitType": "MINUTE",
    "vatRate": 21.0
  }
```

Basically this pricing point means:
After the ride was 4320 minutes long (3 days) we charge 7 EUR per 1440 minutes (1 day)

## Lifecycle of a rental made with TOMP

### Creating a booking
1. Fetch stations using [Stations endpoint](#stations).

   This endpoint returns locations of Donkey stations. It doesn't change that often so it can be cached
   on aggregator's side and refreshed very sporadically.

2. Fetch number of available bikes and parking places in each station using [Available assets endpoint](#available-assets)

   This endpoint returns numbers of available bikes in each station (per bike type, so if there are both ebikes and bikes available
   in particular city it would return separate numbers for how many ebikes and how many bikes are available). Moreover, it returns number
   of available parking spaces per station.

3. Once you know which station you want to rent a bike from start with sygnalizing your booking intention by calling [Planning create endpoint](#planning-create)
   with the station of choice.

   As a result you will receive a list of tentative bookings that are possible to made from given stations. So if there are 2 types of bikes possible
   you will receive 2 booking options. If there is no available bikes in given location you will receive empty list of options.

4. Initiate your booking by calling [Booking create endpoint](#booking-create)

   For this call you will need one of the booking IDs you got in point #3 and information about the customer. As a result you will
   get a booking in bending state.

5. Commit the booking by calling [Booking events endpoint](#commit-booking).

   As a result the returned booking will include `assetAccessData` that contains data needed for unlocking/locking the bike.

   * In case of bluetooth lock `assetAccessData` looks like this:
     ```
      {
        "validFrom": "2020-11-18T20:34:00Z",
        "validUntil": "2020-11-19T21:34:00Z",
        "tokenType": "ekey",
        "tokenData": {
          "ekey": {
            "key": "601212606e241976829f70d68fb6df762eadf17b-7212505588ee8deec3fab3a3268672a8b8f2d4d9-84124c075b52772443a31726d2773fce51df5633-9612a12961227236be1d54e0d8921b88ccd51f98-a812b66a0ee35631aa97ef7186e82316b5cb843d-ba0803db14e989c55ae5",
            "passkey": "5212aebb5ecf2dc30584f176ae1502a5a593d054-52128db9f141a2bbab7d0438adb51af872bc14aa-521290ce0d0af5edd62a4ec23e807b40f7902586-5212daa83f3980a404f9fa84c35e6689a3db7888-5212afbae7176157b44c430fc45b8abab6985da2-5212099f0c1295b9d9f0772e25f5988e7e43282b-521285ff542381d9873f1d9615161ee11e10a6e8-5212c122ff487baad413086f668e4751ea141b95-521212594756d7e7b700ef9f8795470ff36fa8c2-52126180fd04dca4501fcfdd790313461a4f137e-52129a6d1b3b410504de36758d106d2e23c4ab3a-52125c0a42659ad2322234e9f76cf10da47572d3-52122b9486f8f6b9b67e4138f834eac194a33bf8-5212ab9ccf067b6e1ec75a0e37968c31777e2be2-52127308cae4391a1f274f0a3a79359d9015f160-5212725d73a27c611b026ed61259424070f5483d-521272e1dbd8ac2f1b9fee3aca07b324843dba1b-521268c2d7a740fb9b674c24f603716d1b51c61c-5212454ce2099bc267356481f54745c9972b9937-5212ee41987852cb5e1f6645fff22e81f7cc66e6-52123e88469f4960d409e7edd01df77f0104cd77-52126fe5dc1ca0896d9f4e8b0cc9ad8f7ec78c9f-52127b5320dbedac3dd869885f5a2192128bcfc5-52123c72340b0baea1f1e2ce17b8a1277bf8e277-521284eedfc0abc9034216bb2c7a3ecf3f2fb7af-52125fe5907431ed35b3df2cca9fe713b1ed0fb3-5212ba4ffbb5a1bf27de7ebd261276607c4f5ee2-52126df83bee9e32b968f105de641bae5ec25e5c-521290eba99d426a389e15cbf1c5a751c2d2b7de-5212ab2773873fc67a1aa653e6d83035fba82c15-5212daf7730381e66f68a3481cc6ba86fd6e76ca-5212842c80f29bf483a1416fa17cc42a34c1e78d-5212f517c444aacd4283b6ddca7bb6deb7fb8ba8-5212192811e975f3f43d229f3f18205934a79b72-5212800c0b77b349c8f029d9f41ce2f012488c75-52125d6cc3d10b339dbd786ef487d08255f65954-52121074ed44df7501ddc5df2c23895c2b18eeb3-521259d965ba22e775bab14325c66cf8b07840cc-52128b7b44452344d935a38270fc5e05021ca4fb-5212352632a926d076b7b4a840f4c1effcb49319-521275e10e9e6945b0bca9360369b950d50787eb-5212c5ab42ddcfb654c92340319ea9933563b48e-5212f60183ce49f4172cc8777022595924091dae-52127901b9916c7c56a54604518fe36281ac6359-5212f4c74b88389410bca9dc4b5c75345e5499bd-5212736a2f5803117b2c64afb15fd25f87d0dd3b-5212a553382c17f3bd398c0b40084624096f88d1-52128e6561be9fee3da8080b4d221bd417efa629-5212bd67366e0995553b94bd147e3c6095f0cb90-5212261e8e4653baf9501e48a705c38a363f0afa-521272bf52bf214fc59f5b68ff054c86d8b96c21-52120f679ae8cd4a83c57edcb60d6f849163e175-52121d98a30787e3b0abd02d1a5b228ace6ca822-52120878a97fa3fbbb658ea3edb33469fce8100d-52122aceafb2f0575fae5a811ecffb961fb96834-5212027229e1cc6da4fa3937a005c365fad3100f-52125fb525782a5cb2270a2f30ab4f020150a7f9-5212a996e36d11419d77e54c678a0f1918f91b7c-521207eb9074892724a6c5181b080353dcb4c9b5-5212b6afbe0c3746fd95c37af92f3b68d9f6b626-521264d982d77a08e4ae6cf403a9ac6642372f94-5212296c4d628163d8d53ef0b5ae9fbc796f6304-52129cf77685415473b4a139631484c0bdf14cf8-521231d9c8721d048b332d4cbebdb33dd9a23ff9-52124fd8e9e89da2ea3a45ed8535eda854a7a836-52120232e2f3f9eb08a49dfd65e181b51385e3f8-52126e1aa21fb103d048036602545448473e2eb2-5212ce666ec1e504c7e741d2b3f8e7ec64174c6a-5212a9dff98c326649cd841a743ee5841ef67851-52124b34001da153b5943b944f6f2662933b87e6-5212a465b9f76a08ddce1baf3b55db07dc0ea44c-5212d4b684b5709cfa8472b22f4e2db147547c5e-521215261c61b6be187e1354c1596ca66a822e1c-5212f5eb1d517458ede411f0ee7cdbb1c1b30c69-5212cbbe76d8427e211f709d86fb88d4c3e92de3-5212c512455ca1c2e32c8470cd95dd1330887861-52129cfbe382181e041c7f0979c562345423461d-52128064fd8c01a2164fe4f2a09c6f4d0d6ea372-5212bd100b9796ea6e2c722731729c5dd73fa0f8-5212c1016f201839f8c39f4f848fc705c1f88d41-5212daa6d086599ab2ede267a59d0e0891910b25-5212208823621ddde2a13e189c7cc28055925095-5212409941ad51abb755c1c6bd0042c3a144d5b8-5212347802ec4115b6240ec2eb7a8541890e1100-5212cb220a37852c2fb6e2672fac1b62943af65b-521293a7fd7d0a55aeacfe5194d8a13f04e4258b-521275fe61e1c74ac4fd89f162428e0b65580e3c-52129a08b36d5e18c40d19d254cffa60f0111353-5212bc1fc9b66f7d35692fbe5326cbdb9054c5a5-521270b27d8e9bbf8d023a149d1f4d55940e4f7b-52129f4382f004eb7a4660a89f1daca06f7baa09-52129f3c1bdbd7a10f753658b31bf9b10f74f398-521299e75a53e6cb90e5e74edf1ff682d34cc021-521265d8dfc0f66027415f6b1fdd9cde5a31f147-5212c1f248cc980ce484fe64487f2d533e35cdb3-5212922910d1d6ff3203cfaf76b397af84024e96-52121caf22c2f564e5c5ed429f409e7830cc4577-521284e10dc646f90df4ca6e6873c0a7b269616c-52126e946f7121235f37650108d2824f44f706ae-5212c75010f07ff93a20c1577d321727731c63d1-521248b1469f257dadeebc944a3a235fd05ce530-52122151587e00ad462d1529ba95ee9385551581-5212824ec88fb87417a3e0bd2801e5453d3b8c3f-521277b43bd94f43203a7398de4ad1ecba820ba0-52127ae5a14f9b9f5f3c6fed57a5f71e92f782cf-5212387ed1e8189db4346ef9155e25bb3a036b38-52121e4e36096cf240522a5abe0db29695f9ddc5-52125437cddd188774eec06cd5e9c5f33e7abc70-52120e4b5e77559c149ff9a292e9486a6f9855a2-52123c4be88f4c921f5821be49386f5faab84cce-5212168cd732c1409900bd4f33a8e0c030657e4c-5212224302287a0ea0fc48fc2a7f1864dc40b5e9-52122f9d5c30653b284ed7b999045b56fef95da7-521249a1075d438a3fa7b79528e1a0c0996f6483-52122c8bb710cbeb02448a9abaa1abe9ae336bb1-5212179ec52c314175208dd9b97f35894fc07d4f-521252e97293180b851cdf1596ade07e593c37d9-5212499b24090e7a5d1656a07ac05c2cff407484-521257261d8297f374abb4aabde47f27daa43553-5212e9bbdd3fe167b0dabb33d9ae446604db80cd-52127ae3444735a8718dad2cd2ed3203d5757f96-5212c81ecb99e78c53635d4bfe60c87a5ed70bde-5212fe410d44e133420d4e1f952b9efabb3602b0-521262ddad0f7db3ea6e17164269a6d738a51a4a-521235a4423b88127997a7e7bf1f08f584256c00-521219605c08754f84d0cd9db7791a4a86371204-52122683b6a28f14450f3fe84bd0e85a298ddc81-5212e4d6f6e7319014317e0cf4f559c678bbd107-521252bd369d2885f9c398a082f21d1b1a3e72a5-5212afe9e951532e0b7d7bf1cbfff0aa5cada286-52124fc948933dc06d851dfd4b49b90f44edda94-521246c2b42cc87c0983b731bc23df26d9c558d7-5212db88521a849f507cde360a07e113cfd38e26-5212b068fb4b07ad65045f4a1ad3962a5864c670-5212f5c387ec9c2fee66dd7d371ade07058f1963-5212256a1b8ad088926d57ffd7ef5a4a1d7811c7-5212b0b50ca026d03c1b27b7af6ed747b72eb4a0-5212b6b3bf253266dd7cc44b476484615dd95de5-5212cf124d58ba4cce9de5be96b5c0a98e255277-52125fc873352581a5af4a28db8d7232f43657af-5212c1773690b07e311722b411d991762a418ac4-52123da575948d1339ca61e170bb5f0a39423ab0-5212fd9547f4e4e7515e65cc801b8bc3fecf2bb3-521281e3f0decd33cfe64b80fa7f812e9d6a30bc-52125b0e56e158f91e0d0b496db1a6c299283811-5212e3cf23f78ede4dffc658367606652d424eaf-5212999d5a3ffd36197a16482c78db3b54217415-5212cb8e31b49e24fcb7494320e1817c039f8fe0-521292f6353d46db91e882a50a22f3f6fde98381-521291a0652e6855319c9df913796f743c06a184-5212b103be31477e8b9a3e3b15ff20c964f7082a-52126086fadb5ffbbc5991ad84cf4e1b7ea0703d-5212e493b24779b6081599c0fa830d626ded8ed3-5212c671ed60bd494e88c1b6c9fcb310ab287ec8-5212993677ed03fba886fd6479ddb9f121e42e9e-52125ec1abb82e1025c056b8172964eb06ee192a-52125c3bd8d679fc18986df13920b4097cd6ab07-52120084725fda33ad753fbbc25b26dfda2e980f-521288460bd30dcb2bb935429be5913ae411b58c-5212d62e76f7cae57103e5c11303f6ace0306e7e-52128bae9d0e1dbec07e313556c0508d9e946086-5212939a26417c44de73d27d588a2edae87e7c6a-5212fce8dc1a6d3ca7e1d8d0bf467d137974add6-5212bb4fb66fb139c3f7df38994fe713b02ff7d9-5212d80fb34bce15fc3a5ab2e970391f737b3d7f-52129a4183747a2b66e2606ca9c13c48872f4f48-5212df3a721a630768b1da76a79602ceef9e39b8-5212e778f947f890119a14369bcfb9ac55ac1f0e-521246142c7e65b1d49a75e34a14542e49653ba1-52120b2fab094ea365bfdf552a9e9380210f1262-52127dac2e0d45ec57f36c3255d6a10ff1997ad8-5212b3b6c5a4f0e4f4ed81b6b1f73898e6ab6e35-5212621a8403c9e9f5444f1240a87383edd02c9e-52123e3df38aa18f347a0585c50b40bfe8e5ea72-5212fe7b79fe2e8759cd0f47c0f6d9be981f00b7-521292fc868e51f2b112de6b518566c375059eaf-5212fa88d20d34746c89d5969e4d479a0f64a99b-521217502fe91d42854ab5978c1a22d59ef10d68-52127328a5f20b0c881358c1a83c89a920aa7822-52123fa44281e0c1fd45b09f0d24334d52b56202-5212cb928e5e64655fdb6a511a274685c64a793b-521215c48def75302395fd914d756cf4be7f8c17-52126f8d39090ae7658a94c460a5f7cb328a1c3d-52124ce800a6c13764b24588bc05b769365d500f-52124ce31cc2ccbe19a398c899407ee6caef4096-5212b95b7403cafc1346550c5aef20e0f5672065-5212668f4755eec28be98a0d6505e1e3d17f3011-52125e00416e26494b247c10aadaacef455e57e7-5212ff1e2d8bfda245832bb655a24dff7a5129b7-5212776f3d7c931b8245bd8d8487ed385745d3a5-5212e07ebce29a4951563f009eb2f1c45f4f75e8-521206640706ed1997a3f75ec05e1a40811f7922-5212aeb2563f9bde844a5ad6070d53981ef7a8cf-5212bce1f174f9dd05adf987bacd8121e2c707e8-521240258b9ff63e5820cea707ef964c2eca7d7c-52127037679108c732797807ff10707206aee9e7-5212453f05823f43e1506c80acb184af6da6df6c-5212582af4544bc2b31f920b2f6b873bca05d569-52123582544d5c06704bc84b222ff12a16e2d3b6-521244719b55886ef413c97bd1bc5e1b9da4e1ec-5212f61d3341fce645c95eb83c7c624ebc116651-5212566afefb325f848adb67d439739521e9653a-52126313c9bd63036ddf3fef7ab0d233ca9040d4-521277d5c07fc065f435c30bafb88550c7843fe5-5212673eb6839da5e32a964702b74140f2ff22fe-5212a29760d0dab543371df5244d8f8d5b3a8f43-5212dd078be28cf9119e27c6be867f8a58998cc4-5212c5276b92f528a25a41286fee3c31234c9217-5212cc3f12804a99d57e2d897d731dd871688ccc-521281943a8475d1260c53d112e75420c8c5790c-5212611a339f0d0395f434cb72f72f14ddb52cea-52121ddde8dca7e026aa94d918476334f7e9a969-521299a2d9d65a452eae357f85c58477cf19d095-52122aaa33641e7b51482d07e63d6c2c28dc5592-5212a1d0a8fc01df90b1ab0ee852cc104933d5bd-52125dadb7e6d1c050bb6938d01f8349871fa791-521205c42ea03c51e1f6cc47bc6b55473446c881-52127d1b54960c17e2b0a7954eb418cd842be45d-521225f06bb22cb8a4fa5118a5c1736af0f7a9f1-52128c60b707a201027dc80a7dfbd73f9aeaafb3-5212ef8b08369a54bb2893f9e8872a51fabd788d-521296ea44e341bd482ad909c20e9fe472799829-5212148b2ca0ca31bbf2f3d86df37ebeee6fb267-5212f33a1e2df541daf8a35d28fcd2a5bdae89c5-5212852935e82da70207fa2e8d38c1c80f3f34d7-52122f2658595b73ecbcaa9985a1281348fff714-521258e31d08c79655eb8aace1cab312f0fbe277-52120ac3ac7da3d86dfc5e156fea2018a44b29f6-5212d75931dd471d23cdf87b8e8290458af8aad1-52120915693d4cb15636533fa75bdc81c54003d4-52122ac0a74f8b134391df2f7e77aba780bd83ae-5212b36051b059bc8ad0f5c14255d8f814f9d76e-52126d8806b07f2cda422faf36fd1dacf06802d7-521288e1c487fe6620b5d02aac2d6db166446524-5212e34f60df7371e1f9d5a291280dcae72b38e1-5212784939e2f737d9724e97752e5f73a4ac48df-5212c53887963b3ee9b82665fe9330acdb94d50b-5212f2d8c9d862dd1beea41c9d6d8cb66363fee4-5212984c6ee28bc67f75363b42cd859d438eb83e-52125f65d98810fa4c8d0c9e273dadca38592a16-52122a60f186ca72ccb103b6df95ceffa5ff3812-521239360ae39d0d6134ce2bf06a17682b3d9e21-521278a8892b61b9737a4403949a145850c2757a-52120fa1cbccc4649ae5e8a3317ba0fff98b3352-5212db1a7cd58391d84f444af74e4138dff48c91-521270468b83bf14534c57cb2557d7851830244c-5212b6a3ee526660de82c27ab6756d3451f8461e-52123d3e89aff9b5f8980dbaafa7c9b5cf1db0f3-52123a1609c46a9c0ce237a455a5fde885ab42e7-5212a226ef7b95c8491ba2089523eaea4f810395-5212496333b06469644321dcbc2a098b67d1e42e-5212c2abf853547fded597534df0523bb643620e-5212e55eecf19154dfd83372a5bea887638a1c4c-52128b2bd395ffc2992c77500fc23988d44feaa7-5212b843371a959b469f6f761a5734bd8ae8270e"
          },
          "lock": {
            "bdAddress": "E5:89:99:F9:71:C4",
            "deviceName": "AXA:3197AB0A010A19F03652"
          }
        }
      }
     ```

   * In case of online lock `assetAccessData` looks like this:
     ```
       {
         "validFrom": "2020-11-18T20:34:00Z",
         "validUntil": "2020-11-19T21:34:00Z",
         "tokenType": "online",
         "tokenData": {
           "path": "/legs/239fwefJJOQPBGEAZZ23/events"
         }
       }
     ```

### Cancelling the booking

You can cancel your booking by calling [Booking events endpoint](#cancelling-booking) with `CANCEL` event.
Keep in mind that the booking can be cancelled only till first unlock of the bike.

### Unlocking and locking bike
#### Bluetooth lock

For bluetooth lock the mechanism of locking/unlocking the bike consists of following steps:

*Caveat*

`assetAccessData` contains an ekey and a list of commands for that ekey. Those commands have to be used in order and once a command
has been used you can't use an earlier command. SDK doesn't have a way of synchronizing which commands have already been used. Therefore in case
you detect that user is using different installation of the app (user reinstalled the app or logged in using different phone) then you need to ask
our service to refresh the ekey by calling [Refresh ekey endpoint](#refresh-ekey).

Unlocking:
* Using the SDK with the information from `assetAccessData` to unlock the bluetooth lock
* The SDK contacts the lock and initiates unlock action
* Lock pops open
* SDK returns with a success
* Aggregator has to report the unlock using the [Leg events](#trip-execution-leg-events) endpoint

Lock:
* Using the SDK the aggregator's app should initiate lock action
* SDK connects the lock and initiate lock action
* Lock releases the safety mechanism that prevents accidental locking of the lock
* The app should instruct the user to push the lock into locked position
* Once the lock is locked SDK returns with a success
* Aggregator has to report the lock event using [Leg events](#trip-execution-leg-events) endpooint

#### Online Lock

Unlock:
* Initiate the unlock by contacting [Leg events](#trip-execution-leg-events) endpoint
* Our backend starts the unlocking process
* Once we receive an information that the unlocking succeeded we will report it back to aggregator
  with [Leg events webhook](#leg-events-webhook)

Lock:
* Initiate the lock by contacting [Leg events](#trip-execution-leg-events) endpoint
* Aggregator's app should instruct the user to push the lock in locked position
* Once we receive an information that the locking succeeded we will report it back to aggregator
  with [Leg events webhook](#leg-events-webhook)

### Ending rental

When ending the rental the aggregator's app has to make sure that the user is currently with the bike and that
the lock is locked. Then the only thing left to do is use the [Leg events](#trip-execution-leg-events) endpoint to report
leg finish.

After the rental has been finished, it will have an `arrivalTime` field that keeps the timestamp of rental end.

### Fetching final price

Once rental has been finished you can fetch the final price of the booking by fetching [journal entries](#journal-entries)
for particular bookings

```
// REQUEST
  GET /payment/journal-entry?id=FU-lA9P4MRWn1F8QkO8EiQ

// RESPONSE
[
  {
    "amount": 12.0,
    "amountExVat": 9.92,
    "vatRate": 21.0,
    "vatCountryCode": "NL",
    "currencyCode": "EUR",
    "journalId": "FU-lA9P4MRWn1F8QkO8EiQ"
    "journalSequenceId": "1757",
    "details": {
      "estimated": false,
      "parts": [...]
    }
  },

```

## Endpoints
### Cities
This is the only endpoint that is not scoped by particular city path. Therefore the path for this endpoint is
`/api/aggregators/tomp/cities`. Of course the endpoint has to be authorized with api key as all the other endpoints.

It returns the list of all cities where we support TOMP api.

```
GET /api/aggregators/tomp/cities

[
  {
    "city": "Rotterdam",
    "coordinates": {
      "lat": 51.9242897,
      "lng": 4.4784456
    },
    "countryCode": "NL",
    "path": "/api/aggregators/tomp/donkey_rotterdam"
  }
]

```

### Operator
#### Information
```
/REQUEST
GET /operator/information

//RESPONSE
{
  "systemId": "donkey_rotterdam",
  "language": ["en"],
  "name": "Donkey Republic Netherlands",
  "operator": "Donkey Republic Netherlands",
  "url": "https://www.donkey.bike/cities/",
  "purchaseUrl": "https://www.donkey.bike/pricing/",
  "phoneNumber": "+31 85 8885 646",
  "email": "support@donkeyrepublic.com",
  "feedContactEmail": "support@donkeyrepublic.com",
  "timezone": "Europe/Copenhagen",
  "licenseUrl": "https://www.donkey.bike/terms-and-conditions/",
  "typeOfSystem": "VIRTUAL_STATION_BASED",
  "productType": "RENTAL",
  "assetClasses": [
    "BICYCLE",
    "PARKING"
  ]
}
```

#### Stations
```
//REQUEST
GET /operator/stations

//RESPONSE
[
    {
        "stationId": "10811",
        "name": "Vægtergangen II",
        "coordinates": {
            "lng": 12.6389995,
            "lat": 55.6427773
        }
    },
    {
        "stationId": "17572",
        "name": "Flintholm Alle",
        "coordinates": {
            "lng": 12.5021719,
            "lat": 55.6824307
        }
    },
    {
        "stationId": "7434",
        "name": "Tøndergade",
        "coordinates": {
            "lng": 12.5418888,
            "lat": 55.6696496
        }
    }
]
```

#### Available assets
When the amount of available asset type is 0 in given station the entry for that (asset type, station) pair does not appear in the response

```
//REQUEST
GET ../operator/available-assets

//RESPONSE
[
    // Station 10811 doesn't have any available bikes, only parking spots
    {
        "id": "dropoff",
        "assetClass": "PARKING",
        "assetSubClass": "dropoff",
        "sharedProperties": {},
        "stationId": "10811",
        "nrAvailable": 3
    },

    // Station 17572  has 2 parking spots and 1 available bike
    {
        "id": "dropoff",
        "assetClass": "PARKING",
        "assetSubClass": "dropoff",
        "sharedProperties": {},
        "stationId": "17572",
        "nrAvailable": 2
    },
    {
        "id": "bike",
        "assetClass": "BICYCLE",
        "assetSubClass": "bike",
        "sharedProperties": {},
        "stationId": "17572",
        "nrAvailable": 1
    },

    // Station 7434 has 1 bike, 2 ebikes and no available parking spots
    {
        "id": "ebike",
        "assetClass": "BICYCLE",
        "assetSubClass": "ebike",
        "sharedProperties": {},
        "stationId": "17572",
        "nrAvailable": 1
    },
    {
        "id": "bike",
        "assetClass": "BICYCLE",
        "assetSubClass": "bike",
        "sharedProperties": {},
        "stationId": "17572",
        "nrAvailable": 2
    }
]
```

#### Pricing plans
This represents all pricing plans that appear in given city. Since TOMP API doesn't provide a way of saying
which pricing is for which vehicle type then we embed the id of asset type inside the id of the pricing
so that the id of the pricing would be something like `ebike-12311`

```
[
  {
    "planId": "bike-81",
    "name": "Bike Price",
    "description": "Prices for Bikes",
    "isTaxable": false,
    "fare": {
      "estimated": false,
      "parts": [
        {
          "amount": 1.5,
          "units": 15,
          "scaleFrom": 0,
          "scaleTo": 15,
          "scaleType": "MINUTE",
          "currencyCode": "EUR",
          "type": "FLEX",
          "unitType": "MINUTE",
          "vatRate": 21.0
        },
        {
          "amount": 0.5,
          "units": 15,
          "scaleFrom": 15,
          "scaleTo": 30,
          "scaleType": "MINUTE",
          "currencyCode": "EUR",
          "type": "FLEX",
          "unitType": "MINUTE",
          "vatRate": 21.0
        },
        ...
      ]
    }
  },
  {
    "planId": "ebike-252",
    "name": "Ebike Price",
    "description": "Prices for Ebikes",
    "isTaxable": false,
    "fare": {
      "estimated": false,
      "parts": [
        {
          "amount": 2.5,
          "units": 15,
          "scaleFrom": 0,
          "scaleTo": 15,
          "scaleType": "MINUTE",
          "currencyCode": "EUR",
          "type": "FLEX",
          "unitType": "MINUTE",
          "vatRate": 21.0
        },
        {
          "amount": 1.5,
          "units": 15,
          "scaleFrom": 15,
          "scaleTo": 30,
          "scaleType": "MINUTE",
          "currencyCode": "EUR",
          "type": "FLEX",
          "unitType": "MINUTE",
          "vatRate": 21.0
        },
        ...
      ]
    }
  }
]
```

### Plannings
#### Planning create

When station has only one type of vehicles:

```
POST .../plannings?booking-intent=true
{
  "from": {
    "coordinates": {
      "lng": 12.333,
      "lat": 55.123
    },
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
            "name": "Donkey Station"
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
              }
              ....
            ]
          }
        }
      ]
    }
  ]
}
```

When station has both bikes and ebikes available the response contains two booking options

```
//
// REQUEST

POST .../plannings?booking-intent=true
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
            "name": "Donkey Station",
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
            "name": "Donkey Station",
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

##### Possible errors
* There was a problem with some parameters in the request. Example response

  ```
    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
      "errorcode": 2002,
      "title": "Invalid parameters",
      "detail": "/from/stationId is required"
    }
  ```

### Bookings
#### Booking create
Booking should happen immediatelly after the planning produced booking options. Otherwise there is a risk of
someone fetching all the bikes from particular station. At this point booking is in state PENDING.
We don't provide access codes to the bike yet.

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
  "state": "PENDING",
  "legs": [
    {
      "id": "839423832jIFwe",
      "state": "PAUSED",
      "from": {
        "stationId": "123",
        "name": "Donkey Station",
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
        "overriddenProperties": {
          "name": "Speedy"
        }
      },
      "pricing": {.... },
    }
  ]
}
```

##### Possible errors
* Some parameter is invalid
  ```json
    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
      "errorcode": 3002,
      "title": "Invalid parameters",
      "detail": "/id is required, /customer/email is invalid"
    }
  ```

* The station where the booking was drafted for doesn't have any available bikes anymore
  ```json
    HTTP/1.1 410 Bad Request
    Content-Type: application/json

    {
      "errorcode": 3202,
      "title": "Vehicles no longer available",
    }
  ```

* Given user already has an active booking
  ```json
    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
      "errorcode": 3004,
      "title": "This user has an active booking",
    }
  ```

#### Get Booking
Depending on the state of the booking it may or may not include access data for the bike.

```
// REQUEST
GET /bookings/FU-lA9P4MRWn1F8QkO8EiQ

// RESPONSE
200 OK
{
  "id": "FU-lA9P4MRWn1F8QkO8EiQ",
  "state": "PENDING",
  "legs": [
    {
      "id": "839423832jIFwe",
      "state": "PAUSED",
      "from": {
        "stationId": "123",
        "name": "Donkey Station",
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
        "overriddenProperties": {
          "name": "Speedy"
        }
      },
      "pricing": {.... },
    }
  ]
}
```


#### Booking Events
##### Committing booking
```
// REQUEST
POST /bookings/FU-lA9P4MRWn1F8QkO8EiQ/events
{
  operation: "COMMIT"
}

// RESPONSE
200 OK
{
  "id": "FU-lA9P4MRWn1F8QkO8EiQ",
  "state": "CONFIRMED",
  "legs": [
    {
      "id": "839423832jIFwe",
      "state": "PAUSED",
      "departureTime": "2020-11-18T20:34:00Z",
      "from": {
        "stationId": "123",
        "name": "Donkey Station",
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
        "overriddenProperties": {
          "name": "Speedy"
        }
      },
      "assetAccessData": {
        "validFrom": "2020-11-18T20:34:00Z",
        "validUntil": "2020-11-19T21:34:00Z",
        "tokenType": "ekey",
        "tokenData": {
          "ekey": {
            "key": "601212606e241976829f70d68fb6df762eadf17b-7212505588ee8deec3fab3a3268672a8b8f2d4d9-84124c075b52772443a31726d2773fce51df5633-9612a12961227236be1d54e0d8921b88ccd51f98-a812b66a0ee35631aa97ef7186e82316b5cb843d-ba0803db14e989c55ae5",
            "passkey": "5212aebb5ecf2dc30584f176ae1502a5a593d054-52128db9f141a2bbab7d0438adb51af872bc14aa-521290ce0d0af5edd62a4ec23e807b40f7902586-5212daa83f3980a404f9fa84c35e6689a3db7888-5212afbae7176157b44c430fc45b8abab6985da2-5212099f0c1295b9d9f0772e25f5988e7e43282b-521285ff542381d9873f1d9615161ee11e10a6e8-5212c122ff487baad413086f668e4751ea141b95-521212594756d7e7b700ef9f8795470ff36fa8c2-52126180fd04dca4501fcfdd790313461a4f137e-52129a6d1b3b410504de36758d106d2e23c4ab3a-52125c0a42659ad2322234e9f76cf10da47572d3-52122b9486f8f6b9b67e4138f834eac194a33bf8-5212ab9ccf067b6e1ec75a0e37968c31777e2be2-52127308cae4391a1f274f0a3a79359d9015f160-5212725d73a27c611b026ed61259424070f5483d-521272e1dbd8ac2f1b9fee3aca07b324843dba1b-521268c2d7a740fb9b674c24f603716d1b51c61c-5212454ce2099bc267356481f54745c9972b9937-5212ee41987852cb5e1f6645fff22e81f7cc66e6-52123e88469f4960d409e7edd01df77f0104cd77-52126fe5dc1ca0896d9f4e8b0cc9ad8f7ec78c9f-52127b5320dbedac3dd869885f5a2192128bcfc5-52123c72340b0baea1f1e2ce17b8a1277bf8e277-521284eedfc0abc9034216bb2c7a3ecf3f2fb7af-52125fe5907431ed35b3df2cca9fe713b1ed0fb3-5212ba4ffbb5a1bf27de7ebd261276607c4f5ee2-52126df83bee9e32b968f105de641bae5ec25e5c-521290eba99d426a389e15cbf1c5a751c2d2b7de-5212ab2773873fc67a1aa653e6d83035fba82c15-5212daf7730381e66f68a3481cc6ba86fd6e76ca-5212842c80f29bf483a1416fa17cc42a34c1e78d-5212f517c444aacd4283b6ddca7bb6deb7fb8ba8-5212192811e975f3f43d229f3f18205934a79b72-5212800c0b77b349c8f029d9f41ce2f012488c75-52125d6cc3d10b339dbd786ef487d08255f65954-52121074ed44df7501ddc5df2c23895c2b18eeb3-521259d965ba22e775bab14325c66cf8b07840cc-52128b7b44452344d935a38270fc5e05021ca4fb-5212352632a926d076b7b4a840f4c1effcb49319-521275e10e9e6945b0bca9360369b950d50787eb-5212c5ab42ddcfb654c92340319ea9933563b48e-5212f60183ce49f4172cc8777022595924091dae-52127901b9916c7c56a54604518fe36281ac6359-5212f4c74b88389410bca9dc4b5c75345e5499bd-5212736a2f5803117b2c64afb15fd25f87d0dd3b-5212a553382c17f3bd398c0b40084624096f88d1-52128e6561be9fee3da8080b4d221bd417efa629-5212bd67366e0995553b94bd147e3c6095f0cb90-5212261e8e4653baf9501e48a705c38a363f0afa-521272bf52bf214fc59f5b68ff054c86d8b96c21-52120f679ae8cd4a83c57edcb60d6f849163e175-52121d98a30787e3b0abd02d1a5b228ace6ca822-52120878a97fa3fbbb658ea3edb33469fce8100d-52122aceafb2f0575fae5a811ecffb961fb96834-5212027229e1cc6da4fa3937a005c365fad3100f-52125fb525782a5cb2270a2f30ab4f020150a7f9-5212a996e36d11419d77e54c678a0f1918f91b7c-521207eb9074892724a6c5181b080353dcb4c9b5-5212b6afbe0c3746fd95c37af92f3b68d9f6b626-521264d982d77a08e4ae6cf403a9ac6642372f94-5212296c4d628163d8d53ef0b5ae9fbc796f6304-52129cf77685415473b4a139631484c0bdf14cf8-521231d9c8721d048b332d4cbebdb33dd9a23ff9-52124fd8e9e89da2ea3a45ed8535eda854a7a836-52120232e2f3f9eb08a49dfd65e181b51385e3f8-52126e1aa21fb103d048036602545448473e2eb2-5212ce666ec1e504c7e741d2b3f8e7ec64174c6a-5212a9dff98c326649cd841a743ee5841ef67851-52124b34001da153b5943b944f6f2662933b87e6-5212a465b9f76a08ddce1baf3b55db07dc0ea44c-5212d4b684b5709cfa8472b22f4e2db147547c5e-521215261c61b6be187e1354c1596ca66a822e1c-5212f5eb1d517458ede411f0ee7cdbb1c1b30c69-5212cbbe76d8427e211f709d86fb88d4c3e92de3-5212c512455ca1c2e32c8470cd95dd1330887861-52129cfbe382181e041c7f0979c562345423461d-52128064fd8c01a2164fe4f2a09c6f4d0d6ea372-5212bd100b9796ea6e2c722731729c5dd73fa0f8-5212c1016f201839f8c39f4f848fc705c1f88d41-5212daa6d086599ab2ede267a59d0e0891910b25-5212208823621ddde2a13e189c7cc28055925095-5212409941ad51abb755c1c6bd0042c3a144d5b8-5212347802ec4115b6240ec2eb7a8541890e1100-5212cb220a37852c2fb6e2672fac1b62943af65b-521293a7fd7d0a55aeacfe5194d8a13f04e4258b-521275fe61e1c74ac4fd89f162428e0b65580e3c-52129a08b36d5e18c40d19d254cffa60f0111353-5212bc1fc9b66f7d35692fbe5326cbdb9054c5a5-521270b27d8e9bbf8d023a149d1f4d55940e4f7b-52129f4382f004eb7a4660a89f1daca06f7baa09-52129f3c1bdbd7a10f753658b31bf9b10f74f398-521299e75a53e6cb90e5e74edf1ff682d34cc021-521265d8dfc0f66027415f6b1fdd9cde5a31f147-5212c1f248cc980ce484fe64487f2d533e35cdb3-5212922910d1d6ff3203cfaf76b397af84024e96-52121caf22c2f564e5c5ed429f409e7830cc4577-521284e10dc646f90df4ca6e6873c0a7b269616c-52126e946f7121235f37650108d2824f44f706ae-5212c75010f07ff93a20c1577d321727731c63d1-521248b1469f257dadeebc944a3a235fd05ce530-52122151587e00ad462d1529ba95ee9385551581-5212824ec88fb87417a3e0bd2801e5453d3b8c3f-521277b43bd94f43203a7398de4ad1ecba820ba0-52127ae5a14f9b9f5f3c6fed57a5f71e92f782cf-5212387ed1e8189db4346ef9155e25bb3a036b38-52121e4e36096cf240522a5abe0db29695f9ddc5-52125437cddd188774eec06cd5e9c5f33e7abc70-52120e4b5e77559c149ff9a292e9486a6f9855a2-52123c4be88f4c921f5821be49386f5faab84cce-5212168cd732c1409900bd4f33a8e0c030657e4c-5212224302287a0ea0fc48fc2a7f1864dc40b5e9-52122f9d5c30653b284ed7b999045b56fef95da7-521249a1075d438a3fa7b79528e1a0c0996f6483-52122c8bb710cbeb02448a9abaa1abe9ae336bb1-5212179ec52c314175208dd9b97f35894fc07d4f-521252e97293180b851cdf1596ade07e593c37d9-5212499b24090e7a5d1656a07ac05c2cff407484-521257261d8297f374abb4aabde47f27daa43553-5212e9bbdd3fe167b0dabb33d9ae446604db80cd-52127ae3444735a8718dad2cd2ed3203d5757f96-5212c81ecb99e78c53635d4bfe60c87a5ed70bde-5212fe410d44e133420d4e1f952b9efabb3602b0-521262ddad0f7db3ea6e17164269a6d738a51a4a-521235a4423b88127997a7e7bf1f08f584256c00-521219605c08754f84d0cd9db7791a4a86371204-52122683b6a28f14450f3fe84bd0e85a298ddc81-5212e4d6f6e7319014317e0cf4f559c678bbd107-521252bd369d2885f9c398a082f21d1b1a3e72a5-5212afe9e951532e0b7d7bf1cbfff0aa5cada286-52124fc948933dc06d851dfd4b49b90f44edda94-521246c2b42cc87c0983b731bc23df26d9c558d7-5212db88521a849f507cde360a07e113cfd38e26-5212b068fb4b07ad65045f4a1ad3962a5864c670-5212f5c387ec9c2fee66dd7d371ade07058f1963-5212256a1b8ad088926d57ffd7ef5a4a1d7811c7-5212b0b50ca026d03c1b27b7af6ed747b72eb4a0-5212b6b3bf253266dd7cc44b476484615dd95de5-5212cf124d58ba4cce9de5be96b5c0a98e255277-52125fc873352581a5af4a28db8d7232f43657af-5212c1773690b07e311722b411d991762a418ac4-52123da575948d1339ca61e170bb5f0a39423ab0-5212fd9547f4e4e7515e65cc801b8bc3fecf2bb3-521281e3f0decd33cfe64b80fa7f812e9d6a30bc-52125b0e56e158f91e0d0b496db1a6c299283811-5212e3cf23f78ede4dffc658367606652d424eaf-5212999d5a3ffd36197a16482c78db3b54217415-5212cb8e31b49e24fcb7494320e1817c039f8fe0-521292f6353d46db91e882a50a22f3f6fde98381-521291a0652e6855319c9df913796f743c06a184-5212b103be31477e8b9a3e3b15ff20c964f7082a-52126086fadb5ffbbc5991ad84cf4e1b7ea0703d-5212e493b24779b6081599c0fa830d626ded8ed3-5212c671ed60bd494e88c1b6c9fcb310ab287ec8-5212993677ed03fba886fd6479ddb9f121e42e9e-52125ec1abb82e1025c056b8172964eb06ee192a-52125c3bd8d679fc18986df13920b4097cd6ab07-52120084725fda33ad753fbbc25b26dfda2e980f-521288460bd30dcb2bb935429be5913ae411b58c-5212d62e76f7cae57103e5c11303f6ace0306e7e-52128bae9d0e1dbec07e313556c0508d9e946086-5212939a26417c44de73d27d588a2edae87e7c6a-5212fce8dc1a6d3ca7e1d8d0bf467d137974add6-5212bb4fb66fb139c3f7df38994fe713b02ff7d9-5212d80fb34bce15fc3a5ab2e970391f737b3d7f-52129a4183747a2b66e2606ca9c13c48872f4f48-5212df3a721a630768b1da76a79602ceef9e39b8-5212e778f947f890119a14369bcfb9ac55ac1f0e-521246142c7e65b1d49a75e34a14542e49653ba1-52120b2fab094ea365bfdf552a9e9380210f1262-52127dac2e0d45ec57f36c3255d6a10ff1997ad8-5212b3b6c5a4f0e4f4ed81b6b1f73898e6ab6e35-5212621a8403c9e9f5444f1240a87383edd02c9e-52123e3df38aa18f347a0585c50b40bfe8e5ea72-5212fe7b79fe2e8759cd0f47c0f6d9be981f00b7-521292fc868e51f2b112de6b518566c375059eaf-5212fa88d20d34746c89d5969e4d479a0f64a99b-521217502fe91d42854ab5978c1a22d59ef10d68-52127328a5f20b0c881358c1a83c89a920aa7822-52123fa44281e0c1fd45b09f0d24334d52b56202-5212cb928e5e64655fdb6a511a274685c64a793b-521215c48def75302395fd914d756cf4be7f8c17-52126f8d39090ae7658a94c460a5f7cb328a1c3d-52124ce800a6c13764b24588bc05b769365d500f-52124ce31cc2ccbe19a398c899407ee6caef4096-5212b95b7403cafc1346550c5aef20e0f5672065-5212668f4755eec28be98a0d6505e1e3d17f3011-52125e00416e26494b247c10aadaacef455e57e7-5212ff1e2d8bfda245832bb655a24dff7a5129b7-5212776f3d7c931b8245bd8d8487ed385745d3a5-5212e07ebce29a4951563f009eb2f1c45f4f75e8-521206640706ed1997a3f75ec05e1a40811f7922-5212aeb2563f9bde844a5ad6070d53981ef7a8cf-5212bce1f174f9dd05adf987bacd8121e2c707e8-521240258b9ff63e5820cea707ef964c2eca7d7c-52127037679108c732797807ff10707206aee9e7-5212453f05823f43e1506c80acb184af6da6df6c-5212582af4544bc2b31f920b2f6b873bca05d569-52123582544d5c06704bc84b222ff12a16e2d3b6-521244719b55886ef413c97bd1bc5e1b9da4e1ec-5212f61d3341fce645c95eb83c7c624ebc116651-5212566afefb325f848adb67d439739521e9653a-52126313c9bd63036ddf3fef7ab0d233ca9040d4-521277d5c07fc065f435c30bafb88550c7843fe5-5212673eb6839da5e32a964702b74140f2ff22fe-5212a29760d0dab543371df5244d8f8d5b3a8f43-5212dd078be28cf9119e27c6be867f8a58998cc4-5212c5276b92f528a25a41286fee3c31234c9217-5212cc3f12804a99d57e2d897d731dd871688ccc-521281943a8475d1260c53d112e75420c8c5790c-5212611a339f0d0395f434cb72f72f14ddb52cea-52121ddde8dca7e026aa94d918476334f7e9a969-521299a2d9d65a452eae357f85c58477cf19d095-52122aaa33641e7b51482d07e63d6c2c28dc5592-5212a1d0a8fc01df90b1ab0ee852cc104933d5bd-52125dadb7e6d1c050bb6938d01f8349871fa791-521205c42ea03c51e1f6cc47bc6b55473446c881-52127d1b54960c17e2b0a7954eb418cd842be45d-521225f06bb22cb8a4fa5118a5c1736af0f7a9f1-52128c60b707a201027dc80a7dfbd73f9aeaafb3-5212ef8b08369a54bb2893f9e8872a51fabd788d-521296ea44e341bd482ad909c20e9fe472799829-5212148b2ca0ca31bbf2f3d86df37ebeee6fb267-5212f33a1e2df541daf8a35d28fcd2a5bdae89c5-5212852935e82da70207fa2e8d38c1c80f3f34d7-52122f2658595b73ecbcaa9985a1281348fff714-521258e31d08c79655eb8aace1cab312f0fbe277-52120ac3ac7da3d86dfc5e156fea2018a44b29f6-5212d75931dd471d23cdf87b8e8290458af8aad1-52120915693d4cb15636533fa75bdc81c54003d4-52122ac0a74f8b134391df2f7e77aba780bd83ae-5212b36051b059bc8ad0f5c14255d8f814f9d76e-52126d8806b07f2cda422faf36fd1dacf06802d7-521288e1c487fe6620b5d02aac2d6db166446524-5212e34f60df7371e1f9d5a291280dcae72b38e1-5212784939e2f737d9724e97752e5f73a4ac48df-5212c53887963b3ee9b82665fe9330acdb94d50b-5212f2d8c9d862dd1beea41c9d6d8cb66363fee4-5212984c6ee28bc67f75363b42cd859d438eb83e-52125f65d98810fa4c8d0c9e273dadca38592a16-52122a60f186ca72ccb103b6df95ceffa5ff3812-521239360ae39d0d6134ce2bf06a17682b3d9e21-521278a8892b61b9737a4403949a145850c2757a-52120fa1cbccc4649ae5e8a3317ba0fff98b3352-5212db1a7cd58391d84f444af74e4138dff48c91-521270468b83bf14534c57cb2557d7851830244c-5212b6a3ee526660de82c27ab6756d3451f8461e-52123d3e89aff9b5f8980dbaafa7c9b5cf1db0f3-52123a1609c46a9c0ce237a455a5fde885ab42e7-5212a226ef7b95c8491ba2089523eaea4f810395-5212496333b06469644321dcbc2a098b67d1e42e-5212c2abf853547fded597534df0523bb643620e-5212e55eecf19154dfd83372a5bea887638a1c4c-52128b2bd395ffc2992c77500fc23988d44feaa7-5212b843371a959b469f6f761a5734bd8ae8270e"
          },
          "lock": {
            "bdAddress": "E5:89:99:F9:71:C4",
            "deviceName": "AXA:3197AB0A010A19F03652"
          }
        }
      },
      "pricing": {.... },
    }
  ]
}
```

##### Cancelling booking
```
// REQUEST
POST /bookings/FU-lA9P4MRWn1F8QkO8EiQ/events
{
  operation: "CANCEL"
}

// RESPONSE
204 No Content
```

### Legs
#### Fetch leg

Endpoint that provides current information about the rental (leg)

```
// REQUEST
GET /legs/239fwefJJOQPBGEAZZ23/

// RESPONSE
200 OK

{
    "id": "YzkwOWMwOGQwMy0zNjI4LWJpa2UtMS0w",
    "state": "PAUSED",
    "departureTime": "2021-03-01T13:36:45Z",
    "arrivalTime": "2021-03-01T15:36:45Z" // "arrivalTime" only there for finished bookings
    "from": {
      "coordinates": {
        "lat": 51.9226929,
        "lng": 4.4975173
      },
      "stationId": "3628",
      "name": "Donkey Station"
    },
    "to": {                           // "to" only returned for finished bookings
      "coordinates": {
        "lat": 51.88222,
        "lng": 4.7275173
      },
      "stationId": "3628",
      "name": "Donkey Barn"
    },
    "asset": {
      "id": "7443",
      "overriddenProperties": {
      }
    },
    "assetType": {
      "id": "bike",
      "assetClass": "BICYCLE",
      "assetSubClass": "bike",
      "sharedProperties": {}
    },
    "pricing": {... },
}
```

### Trip Execution (Leg Events)
#### Locking and unlocking
For locking/unlocking we use following events:

* `PAUSE` - lock
* `SET_IN_USE` - unlock

The request also has to include the location of the event as shown below.

This endpoint has different meanings based on the type of the lock
* bluetooth lock - it is used to just report the occurence of lock or unlock
* online lock - it is used to initiate the process of locking/unlocking

```
// REQUEST
POST /legs/239fwefJJOQPBGEAZZ23/events

{
  "time": "2020-11-18T20:34:00Z",
  "event": "SET_IN_USE",
  "asset": {
    "id": "bike-12331",
    "overriddenProperties": {
      "location": {
        "coordinates": {
          "lng": 12.5021719,
          "lat": 55.6824307
        }
      }
    }
  }
}

//RESPONSE
204 No Content
```

#### Finishing rental
The difference between locking/unlocking events described above and the finish leg event
is that for the finish event we just need confirmation that at the moment of the event the
user was close to the bike (`withLockConnection` flag) and that the lock is locked (`isLocked` flag).

```
// REQUEST
POST /legs/239fwefJJOQPBGEAZZ23/events

{
  "time": "2020-11-18T20:34:00Z",
  "event": "FINISH",
  "asset": {
    "id": "bike-12331",
    "overriddenProperties": {
      "location": {
        "coordinates": {
          "lng": 12.5021719,
          "lat": 55.6824307
        }
      },
      "meta": {
        "withLockConnection": true,
        "isLocked": true
      }
    }
  }
}

//RESPONSE
204 No Content
```
##### Possible errors
* When lock is unlocked
  ```json
    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
      "errorcode": 4004,
      "title": "Operation is illegal",
      "detail": "Lock has to be locked"
    }
  ```

* When user is not with vehicle
  ```json
    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
      "errorcode": 4004,
      "title": "Operation is illegal",
      "detail": "User has to be with vehicle"
    }
  ```

* When bike is not inside a station
  ```json
    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
      "errorcode": 4004,
      "title": "Operation is illegal",
      "detail": "Rental has to end inside a dropoff location"
    }
  ```

* When leg is already finished
  ```json
    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
      "errorcode": 4004,
      "title": "Operation is illegal",
      "detail": "Leg is finished"
    }
  ```

* Invalid parameters - details of invalid parameters will be described in the `detail` property. Possible errors are missing meta parameters, invalid or missing coordinates
  ```json
    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
      "errorcode": 4002,
      "title": "Invalid properties",
      "detail": "/asset/overriddenProperties/meta is missing"
    }
  ```

* Leg is not found
  ```json
    HTTP/1.1 404 Not Found
    Content-Type: application/json

    {
      "errorcode": 4001,
      "title": "Leg not found"
    }
  ```

#### Refresh ekey
In case you detect that the user changed the installation of the app that they are using (they logged in on different phone or reinstalled the app) then for bluetooth locks
you need to ask our service to refresh the ekey.

```
// REQUEST
POST /legs/239fwefJJOQPBGEAZZ23/events

{
  "time": "2020-11-18T20:34:00Z",
  "event": "ASSIGN_ASSET",
  "asset": {
    "id": "bike-12331",
    "overriddenProperties": {
      "location": {
        "coordinates": {
          "lng": 12.5021719,
          "lat": 55.6824307
        }
      }
    }
  }
}

//RESPONSE
204 No Content
```

Once you get a successful response, data in get leg or get booking endpoint will contain new ekey.


### Payment
#### Journal Entries
The journal entries returns all invoiced items for particular aggregator and city.

There are few options of filtering jornal entries by passing query parameters
* `id` - String - id of the booking for which journal entries should be returned
* `from` and `to` - datetime in ISO8601 format (example `"2021-03-23T14:24:31Z"`) -
  to select a timeframe for which journal entries should be returned
* `offset` and `limit` - integer - basic pagination

There are 3 types of journal entries that you can see in the response:
* Regular booking charges - those are the charges that the rider is charged for renting the bike.
  Those journal entries will have a `fare` constrcut in details.

* Fines - if we have to charge any fine it will show up here. Details of such journal entry
  will contain extra costs representation with category `"FINE"`.

* Refunds - whenever we issue a refund of any of the charges above. Details of refund journal entry
  will conain extra costs representation with category `"REFUND"`. Moreover, we will include identified for a charge
  that is being refunded in `meta` field (as shown below in the example).

We have 2 fields that identify journal entries:
* `journalId` - it represents a booking id so even when there is a list of multiple journal entries each one can be connected to a booking
* `journalSequenceId` - identifier of particular journal entry

```
GET /payment/journal-entry
[
  // Regular charge
  {
    "amount": 12.0,
    "amountExVat": 9.92,
    "vatRate": 21.0,
    "vatCountryCode": "NL",
    "currencyCode": "EUR",
    "journalId": "YzkwOWMwOGQwMy0zNjI4LWJpa2UtMS0w"
    "journalSequenceId": "1757",
    "usedTime": 125, // How long was given leg in seconds, only shown for regular booking charges
    "details": {
      "estimated": false,
      "parts": [
        {
          "amount": 15.0,
          "units": 15,
          "scaleFrom": 0,
          "scaleTo": 15,
          "scaleType": "MINUTE",
          "currencyCode": "EUR",
          "type": "FLEX",
          "unitType": "MINUTE",
          "vatRate": 21.0
        },
        {
          "amount": 100.0,
          "units": 1440,
          "scaleFrom": 15,
          "scaleType": "MINUTE",
          "currencyCode": "EUR",
          "type": "FLEX",
          "unitType": "MINUTE",
          "vatRate": 21.0
        }
      ]
    }
  },

  // Fine
  {
    "amount": 10.0,
    "amountExVat": 8.26,
    "vatRate": 21.0,
    "vatCountryCode": "NL",
    "currencyCode": "EUR",
    "journalId": "YzkwOWMwOGQwMy0zNjI4LWJpa2UtMS0w"
    "journalSequenceId": "1955",
    "details":  {
      "amount": 10.0,
      "amountExVat": 8.26,
      "vatRate": 21.0,
      "vatCountryCode": "NL",
      "currencyCode": "EUR",
      "category": "FINE",
      "description": "Lost Bike fee for one bike, fine_id: 454"
      }
  },

  // Refund
  {
    "amount": -5.0,
    "amountExVat": -4.13,
    "vatRate": 21.0,
    "vatCountryCode": "NL",
    "currencyCode": "EUR",
    "journalId": "YzkwOWMwOGQwMy0zNjI4LWJpa2UtMS0w"
    "journalSequenceId": "1973",
    "details": {
      "amount": -5.0,
      "amountExVat": -4.13,
      "vatRate": 21.0,
      "vatCountryCode": "DK",
      "currencyCode": "DKK",
      "category": "REFUND",
      "description": "Refund",
      "meta": {
        "refund_for": {
          "journalId": "YzkwOWMwOGQwMy0zNjI4LWJpa2UtMS0w"
          "journalSequenceId": "1955",
        }
      }
    }
  }
]
```

## Webhooks

The webhooks are a way for DonkeyRepublic to report changes that happen in the rental
(like changes of lock state of the online lock, assigning different bike to the user by support etc)
back to the aggregator.

### Leg Events Webhook

Communicates asynchronous changes to the rental
* `SET_IN_USE` - the online lock has been unlocked
* `PAUSE` - the online lock has been locked
* `ASSIGN_ASSET` - our support assigned a new bike to the user
  In such a case information on the data needed to access the new bike can be obtained by
  calling [Fetch leg](#fetch-leg) endpoint (as unfortunatelly the leg event call does not have
  this information in TOMP)
* `FINISH` - rental was ended on our side (due to support query etc)

```
// REQUEST
POST /legs/239fwefJJOQPBGEAZZ23/events

{
  "time": "2020-11-18T20:34:00Z",
  "event": "SET_IN_USE",
  "asset": {
    "id": "bike-12331"
  }
}

//EXPECTED RESPONSE
204 No Content
```

### Additional costs

There are 2 cases when we will trigger that webhook:
* Whenever there is some penalty/fine added to the booking due to some incorrect usage of the bike. In such case the category would be `"FINE"`

* Whenever there is some refund issued for the booking after the booking has been finished. This can be both for the initial charge of the booking or for
subsequent fines. The category is `"REFUND"`. In case of refunds amounts are negative. Moreover, refunds will contain information about which journal entry is being
refunded as seen in [Journal Entries endpoint](#journal-entries)

```
POST /payment/{booking-id}/claim-extra-costs
{
  "amount": 1.5,
  "amountExVat": 1.24,
  "vatRate": 21.0,
  "vatCountryCode": "NL",
  "currencyCode": "EUR",
  "category": "FINE" // possible categories: ["REFUND", "FINE"]
  "description": "Lost Bike fee for one bike, fine_id: 454"
}

// EXPECTED RESPONSE
200 OK
{
  "amount": 1.5,
  "amountExVat": 1.24,
  "vatRate": 21.0,
  "vatCountryCode": "NL",
  "currencyCode": "EUR",
}
```

### Testing webhooks
Disclaimer: Testing endpoints are not part of official TOMP implementation and are provided only to help with testing.

#### Updating bike lock state and location
For testing ebikes we introduced an endpoint that makes it possible to change state and location of the bike. In case lock state changes
it will also trigger a `SET_IN_USE` or `PAUSE` webhook. This endpoint works for ebike bookings only.

```
REQUEST
POST /api/aggregators/tomp/testing/bike_state
{
  "leg_id": "YTg4ZTdiZGE3NDk0YWZjNTRkZTctMTg1MC1lYmlrZS0xLTA",
  "bike_state": {
      "latitude": 52.111,
      "longitude": 12.100,
      "locked": false
}
```

### Triggering state changes of the leg
There are some situations where state of the leg will change and it is not triggered by your interaction with our TOMP API:
* Support has to cancel a leg for given rider
* Support has to finish a leg for given rider
* Support assigns different bike to a leg

Those state changes are communicated by leg webhooks. You can trigger them by calling endpoint described below with following actions:
* `"FINISH"` - will finish given leg
* `"CANCEL"` - will trigger cancellation of the leg
* `"ASSIGN_ASSET"` - will assign a different bike to given leg. It will select a bike of the same type from a station where a leg was started. Therefore you have to make
  sure that you test it with legs that started from a station that has other available bike of given type. In case there is no available bike to switch to then you will get an error.

Keep in mind that this endpoint can be triggered on ongoing legs only, otherwise you will get an error

```
REQUEST
POST /api/aggregators/tomp/testing/leg_action
{
    "leg_id": "OWFkZWE3MDhhN2Q4Njc4ODJjZDItMTI2MTEtYmlrZS0xLTA",
    "leg_action": "FINISH" // or "CANCEL" or "ASSIGN_ASSET"
}
```

#### Extra costs
We introduced a way of triggering an extra costs webhooks via an API call. Those API will work only in our staging environment.

1. Triggeing a fine extra costs event

```
REQUEST
POST /api/aggregators/tomp/testing/extra_costs
{
    "booking_id": "booking_id",
    "category": "FINE"
}

RESPONSE
204 No Content
```

This endpoint will charge a FINE on a selected booking. Keep in mind that the booking has to be finsihed for this to work, can't be used on ongoing booking.

2. Triggering a refund event

```
REQUEST
POST /api/aggregators/tomp/testing/extra_costs
{
    "booking_id": "booking_id",
    "category": "REFUND",
    "journal_sequence_id": "2230859"
}

RESPONSE
204 No Content
```

This endpoint will refund one of journal entries. It does a refund for full amount of given journal entry. Keep in mind that if given entry has been already refunded then
the result of this call will be noop.
