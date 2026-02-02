In credit life we get live products from studio, whereas in life-service through lifeProductMaster.
In credit life then again to generate validation map we call hub (external call) and then the score = 0.0 is used as valid. But in life service we have two ways of creating validation map newImplementation and oldImplementation. 
New Implementation -> Does call to studio and do the filtering, and then further db queries on rider after that non null score products are removed and the nearest base match products are returned.
In old Implementation -> validation and filtering is totally dependent on DB queries.

Then in life-service validationMap is returned back in /request api response. Which is then passed as keylist in the payload of /results api. Platform-service then one by one hits the productCode to vertical-service.
Here again initially same validation is done but this time rather than for all products, only the productCode for which the API is hit. It has some calculation logic also based on ULIP. And after that the insurer call is done to get the premium.

In credit life in the same API itself, when we get the validation map, we create the premium request scope, again the IH is called to set add ons. After that the premiumrequest call is made to IH, the response is then transfomed and mapped back.

In credit life everything happens in that one /quotes api
