## Example query focused

```json
{
   // meta information, such as transported via headers (or "$headers"?)
   "$": {
      "single": true, // or "maybe" or false,
      "count": "exact" // or "planned" or "estimated"
   },
   "type": "query", // optional, defaults to "query"
   "version": "1.0.0", // optional, defaults to latest
   "from": "products",
   "select": [
      "id",
      "name",
      "price",
      {
         // instead of string to add aliasing info
         // as base idea for postgrest advanced select features
         "description": {
            "type": "alias",
            "as": "d"
         }
      },
      {
         "metadata": {
            "type": "json",
            "path": "$.theme" // example, keep in mind the JSON operators available in postgres, maybe there is a better way to express this
         }
      }
   ],
   // embedded objects
   "with": {
      "categories": {
         "as": "c", // optionally aliasing, just an idea
         "select": ["id", "name"],
         "where": {
            "name": {
               "$eq": "electronics"
            }
         }
      }
   },
   "where": {
      "name": {
         "$like": "*iPhone*", // maybe also "%" since we only focus on SQL
         "$in": ["iPhone", "Android"]
      },
      "price": {
         "$gt": 100,
         "$lt": 200
      },
      "category": {
         "$neq": "electronics"
      },
      "$or": {
         "name": {
            "$like": "*Android*"
         }
      }
   },
   "order": {
      "price": "asc",
      "name": "desc"
   },
   "limit": 10,
   "offset": 0
}
```
