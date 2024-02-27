VM IP: 10.15.15.253
DB IP: 10.15.15.197
WBLFramework Production deployment.

Execute these Queries on the WBLFramework Database:-

1. db.getCollection("Batch").updateOne({
   "batchCode":"BATCHCODE002"
},{
   "$set":{
      "mPartId":"927e3e26-9d61-4bd2-a891-775dbf7c1e6b",
      "partId":"69c3ee04-0f47-4655-8da9-7b535fb7c240",
      "partName":"FY",
      "partCompleteName":"First Year"
   }
})