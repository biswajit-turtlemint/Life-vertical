- In credit life we get live products from studio, whereas in life-service through lifeProductMaster.
- In credit life then again to generate validation map we call hub (external call) and then the score = 0.0 is used as valid. But in life service we have two ways of creating validation map newImplementation and oldImplementation. 
- New Implementation -> Does call to studio and do the filtering, and then further db queries on rider after that null score products are removed and the nearest base match products are returned.
- In old Implementation -> validation and filtering is totally dependent on DB queries.

- Then in life-service validationMap is returned back in /request api response. Which is then passed as keylist in the payload of /results api. Platform-service then one by one hits the productCode to vertical-service.
- Here again initially same validation is done but this time rather than for all products, only the productCode for which the API is hit. It has some calculation logic also based on ULIP. And after that the insurer call is done to get the premium.

- In credit life in the same API itself, when we get the validation map, we create the premium request scope, again the IH is called to set add ons. After that the premiumrequest call is made to IH, the response is then transfomed and mapped back.

- In credit life everything happens in that one /quotes api

## Solution
- We can use credit-life archtecture as base and make our calls to HUB. But we will need a IH team for this.
- Riders in TM also for life comes through DB. But for credit-life it comes from HUB. 
- We are planning to move to new implementation on minterprise where all calls will be through IH, but if a client is not having HUB. Then should we also have a DB based filtering also which means old Implementation.
- After fetching data from IH, validation logic and filtering will be same as life-service. If something is already getting covered in IH end we will skip that filtering logic. Need to discuss with IH team.
- Even after calling IH for life in new implementation we perform further filtering and validation in code. We need to check all and see with IH if this can be handled or should we add in code.
