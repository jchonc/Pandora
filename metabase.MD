#Some hypothesis around Metabase

##Background
* Both MSSQL and PostgreSQL support some JSON based operations.
* Most of the information in a MetaDataSet is load/manipulate/saved as a whole.
* JSON will allow us to hide the schema differences among different tenants from the relational database.
* Most of the CURD operation will be API based

##Breakdown
* Dictionary
* How an entity get fetched and saved
* Expression in searh/filter
* Expression in form engine
* Audit log

##Dictionary
* We should allow multiple root tables per module to accomodate infection and weaken the sterotype of everything is just file.
* We might officially support multiple links between the same entity groups (maybe via lookup)
* We still need to ensure on per entity level, a tree structure can be pulled out of the forest/network. 
* Need to be able to distinguish physical entity/elements vs. JSON entity/elements

##CRUD Operations
* Assuming we use EFCore 2.0 as the main data access means, by default, a composite JSON field will appear as a string property and we could create a twin ExpandoObject property to expose the inside content. 
```
    public string DetailsJson { get; set; }
    public dynamic Details { 
        get 
        {
            if (_details == null)
            {
                if (!String.IsNullOrEmpty(DetailsJson))
                {
                    _details = JsonConvert.DeserializeObject<ExpandoObject>(DetailsJson);
                }
                else
                {
                    _details = new ExpandoObject();
                }
            }
            return (dynamic)_details;
        }
    }
```
Then we can access it like usual (this will be mostly of the business logic code, i.e. close file)
```
    firstInc.Details.injured = true;
    firstInc.Details.injuryNatures = "Some other values";
```
All we need to do is make sure we serialize it back before save
```
    DetailsJson = Newtonsoft.Json.JsonConvert.SerializeObject(_details);
```
* For the normal fetch operation, we can just serialize the object as-is. An optional ContractResolver can be used to carry out the forbidden fields requirements.
```
    ...
    public class SafeJsonContractResolver: DefaultContractResolver
    {
        private readonly string[] _forbiddenAttrNames;

        public SafeJsonContractResolver()
        {
            _forbiddenAttrNames = null;
        }

        public SafeJsonContractResolver(string[] forbiddenAttrNames)
        {
            _forbiddenAttrNames = forbiddenAttrNames;
        }

        protected override IList<JsonProperty> CreateProperties(Type type, MemberSerialization memberSerialization)
        {
            IList<JsonProperty> properties = base.CreateProperties(type, memberSerialization);
            if (_forbiddenAttrNames != null && _forbiddenAttrNames.Count() > 0)
            {
                properties = properties.Where(p => !_forbiddenAttrNames.Contains(p.PropertyName)).ToList();
            }
            return properties;
        }
    }

    ...
    
    public string AsJson()
    {
        //Pretent the list holds disallowed attributes
        string[] forbiddenAttrNames = new string[2] { "bankAccount", "TenantId" };
        return Newtonsoft.Json.JsonConvert.SerializeObject(
            this,    
            Newtonsoft.Json.Formatting.Indented,
            new Newtonsoft.Json.JsonSerializerSettings { 
                ContractResolver = new SafeJsonContractResolver(forbiddenAttrNames) 
            }
        );    
    }
```


* The way back is much more treacherous though. So far there is a [JSON patch](https://tools.ietf.org/html/rfc6902) but it's not very front-end friendly.  Recently there is another [JSON Merge Patch](https://tools.ietf.org/html/rfc6902) and while it's much welcomed by developer the toolkit is not complete yet. On AspNetCore side, Microsoft has provided an [JsonPatchDocument](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.jsonpatch.jsonpatchdocument?view=aspnetcore-2.0) to carry out the first one and the demand for the second one is [there](https://github.com/aspnet/Home/issues/2436). So far the most elegatn way is from [JsonMergePath](https://github.com/Morcatko/Morcatko.AspNetCore.JsonMergePatch). -- It reads the incoming JSON and generates the ops and internally still uses JsonPatchDocument to execute the operations.  
```
    [HttpPatch]
    [Route("api/incidents/{id}")]
    [Consumes(JsonMergePatchDocument.ContentType)]
    public object PutIncident([FromQuery] string id, [FromBody]JsonMergePatchDocument<Incident> patch)
    {
        const string firstIncidentId = "22C0745E-94D4-4FBA-B9AE-2932C4BD4D10";
        using (var context = new BusinessContext())
        {
            var firstInc = context.Incidents.Find(Guid.Parse(firstIncidentId));
            if (firstInc != null)
            {
                try
                {
                    firstInc.ApplyPatch(patch));
                    firstInc.Validate();
                    context.SaveChanges(true);
                }
                catch(Exception ecpt)
                {
                    return BadRequest(ecpt.Message);
                }
            }
            else
            {
                return NotFound($"Incident ${id} not found");
            }
        }
        return Ok();
    }
```
In the above code, the request post body is just the full JSON, thus very much JavaScript friendly. Again, internally same ContractResolver could be used to ensure some elements are not touched (tenantId, id, key fields, forbidden fields ... etc.)
```
    public void ApplyPatch(JsonMergePatchDocument patch)
    {
        //Pretent the list holds disallowed attributes
        string[] forbiddenAttrNames = new string[2] { "id", "TenantId", "salary" };
        patch.ContractResolver = new SafeJsonContractResolver(forbiddenAttrNames);
        //Casting generic to local concrete class at runtime is not very easy
        Type thisPatchType = typeof(JsonMergePatchDocument<>).MakeGenericType(this.GetType());
        thisPatchType.GetMethod("ApplyTo").Invoke(patch, new object[] { this });
        SaveDetails();
    }
```

* Validation

I haven't touched this area yet. There are several possible leads however
    - Newtonsoft seems to have a perpetual implementation of [JSON schema](https://www.newtonsoft.com/jsonschema). If we can generate the proper schema from meta dictionary then it could be used as static validation.
    - I suspect for dynamic valiation (similar to CheckMandatory in RL6) we'll have to write our own code anyway since [the standard JSON schema validation](https://tools.ietf.org/html/draft-handrews-json-schema-validation-00#page-12) does not really accomodate any calculations/expressions in required flag.


##Search & Filter

* This is the part I got stuck right now. Neither database provides an elegant solution for this.

* JSON_VALUE can be used to extract specific attributes from JSON fields, which could further be allowed to map to calcualted column or even index upon.  This will enable those values to be included in the final flated out result set.

* However the expression (the where fragment) is very hard. The default JSONPATH support none of the expression and array filtering, unless fall back to the JSON_VALUE approach, but then it's not array and nestling friendly. 

* ElasticSearch might be the only way to support this though. 

##Support for Form Engine

* Once brought to client side, this is just a JSON object. Though we might want to look deeper into it to see how dynamic expression could be binded to it.  So far I am reading/playing on [mobx.extendObservable](https://mobx.js.org/refguide/extend-observable.html) and if [mobx-state-tree](https://github.com/mobxjs/mobx-state-tree) could be build on top of a dynamic schema. 

##Misc.

* We have used [StringTemplate4](https://github.com/antlr/antlrcs) to generate notification email and I think this could be a very good candidate to carry out summary and form letter needs. 

* I would recommend we retire HTML/rich fields if possible, given they are very hard to query. 
