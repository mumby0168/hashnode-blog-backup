## Paging in Azure Cosmos DB

# Foreword

Cosmos DB is a popular NoSQL database offering from Microsoft's cloud provider Azure. It offers an extremely scalable, super-fast, and highly available platform as a service database. All data is stored as JSON, in a logical grouping called a container.  Containers can scale independently of one another and configure independent partitioning strategies.

> If containers, partitioning, and NoSQL databases are new to you, I would recommend reading the Cosmos DB documentation found [here](https://docs.microsoft.com/en-us/azure/cosmos-db/). 

# Paging use cases

Why do you need to page data? There are many cases for paging data. But I believe there are two examples that best explain the different paging options provided by Cosmos DB. However, these use cases apply to all databases.

The first is a common scenario you've probably used in social media, _infinite scrolling_. Think of Twitter, for example, you load some tweets then scroll down the feed. As you read the first set of tweets more are loaded for you to read, and so on. This is a form of paging. You can scroll back up the tweets but it won't re-query the database for those tweets again. The tweets are already loaded onto your device so you just scroll back up, and they are still there. The fact it does not re-query the database is important in this example. If you think of this from an implementation point of view, this is like a marker, and as you read the next page that marker moves. However, this "marker" always moves forward's never backward in this example. This is important to remember for later.

The second example I like to use, is a typical table, in an application that has common features. Such as _next_ and _previous_ buttons, a dropdown to allow you to specify the maximum number of results returned, and so on. Something like the table below taken from [MD Bootstraps docs](https://mdbootstrap.com/docs/b4/jquery/tables/pagination/).

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1642187327324/YrJZi8gdY.png)

This allows a user to have random access to any page, moving back and forth through the pages, potentially changing the sorting and filtering of the data and the page size of the given results. This as you can imagine, requires the database do a lot more work, but also offers a lot more flexibility in terms of what you can offer to a user.

# The Cost of Paging

Paging in any database is an expensive feature in terms of resources that will be used up by a database if it is frequently paging data. In the random-access scenario, a lot of implementations rely on either _skip_ and _take_, or _limit_ and _offset_ approaches. A common example of this could be making a query to retrieve page 5 of some data with a page size of 25. You would have a query that skips the first 100 records but limits the returned records of the query to 25 records. This is supported by Cosmos DB and many other databases but this means the database has to skip 100 rows which it still has to process, it can't just jump to a point in time or to a cursor for example. When you translate this into Cosmos DB terms this skip will add to the number of RUs that your query costs. In a lot of cases making some queries very expensive if run frequently.

> What are "RU's"? RU's or request units are the cost of an operation, think of this as the currency of a Cosmos database read more on this [here](https://docs.microsoft.com/en-us/azure/cosmos-db/request-units).

Taking a step back to the infinite scroll scenario described earlier if you were to implement something like this in Cosmos DB there is an awesome feature that can be utilized. Cosmos DB implements continuation tokens for its paging operations, this is a stateless marker in a database of where a current query is at. This means when provided back to Cosmos DB it can pick up right where you left off with a mostly consistent RU charge. The beauty of these being stateless means you could make a query and then not come back and request the next page of data for a few hours or days and it will pick up just where you left off. 

> Want to know more about continuation tokens? Check out the  [Cosmos DB docs](https://docs.microsoft.com/en-us/azure/cosmos-db/sql/sql-query-pagination#continuation-tokens)!

# Paging Code Samples

The following code examples come in the form of .NET 6 Web API projects, both of which can be found [here](https://github.com/mumby0168/cosmos-db-paging). The first example uses the raw  [.NET Cosmos SDK](https://github.com/Azure/azure-cosmos-dotnet-v3) and the second uses the [Azure Cosmos Repository](https://github.com/IEvangelist/azure-cosmos-dotnet-repository). Some of the code from these samples will be excluded in this post. This is to focus on paging implementation using both libraries.

> The Azure Cosmos Repository wraps the .NET Cosmos SDK, aiming to take away some of the ceremony involved with the .NET SDK when working with containers.

## Paging in Cosmos DB using the .NET SDK 

### The _skip_, _take_ approach

The first code example is an API endpoint that allows a client to page through a container with a set of people records.

```c#
app.MapGet("/skipTake", async (
    [FromServices] CosmosClient client,
    [FromQuery] int pageNumber,
    [FromQuery] int pageSize) =>
{
    var container = await GetPeopleContainer(client);

    QueryRequestOptions queryOptions = new()
    {
        MaxItemCount = pageSize
    };

    IQueryable<Person> query = container
        .GetItemLinqQueryable<Person>(requestOptions: queryOptions)
        .Where(x => x.PartitionKey == nameof(Person))
        .Skip(pageSize * (pageNumber - 1))
        .Take(pageSize);

    var (items, charge, _) =
        await GetAllItemsAsync(query, pageSize);

    return new
    {
        requestUnits = charge,
        count = items.Count,
        people = items,
    };
});
```

> In the above example, there is a method called `GetAllItemsAsync(...)` this does a lot with the SDK to properly get the results and map them into a list. This can be found [here](https://github.com/mumby0168/cosmos-db-paging/blob/904363b9d74d0dc8f49269f92d9575315c453360/CosmosDbSdkPagingSample/Program.cs#L42)

This example work's as expected making a request such as `GET https://{{host}}:{{port}/skipTake?pageSize=25&pageNumber=1` returns 25 people and costs `3.6 RUs`. See the response below.

```json
{
  "requestUnits": 3.6,
  "count": 25,
  "people": //excluded for brevity
}
```

3.6 RUs isn't certainly not an expensive query in Cosmos DB terms. However, when you query for page 10 with a request such as `GET https://{{host}}:{{port}/skipTake?pageSize=25&pageNumber=10`. This request costs nearly `10 RUs`, see below.

```json
{
  "requestUnits": 9.86,
  "count": 25,
  "people": //excluded for brevity
}
```

This links back to some of the problems with the _skip_, _take_ approach explained earlier. You can imagine as the page size increases and you begin to query for page 50 or 100 on a larger data set, this charge keeps on increasing.

### Paging with continuation tokens

The next example shows how the infinite scroll scenario can be implemented using the .NET SDK.

```csharp
app.MapGet("/tokenBased", async (
    HttpContext context,
    [FromServices] CosmosClient client,
    [FromQuery] int pageSize) =>
{
    string? continuationToken = 
        context.Request.Headers["X-Continuation-Token"];
        
    var container = await GetPeopleContainer(client);

    QueryRequestOptions queryOptions = new()
    {
        MaxItemCount = pageSize
    };

    IQueryable<Person> query = container
        .GetItemLinqQueryable<Person>(
            requestOptions: queryOptions, 
            continuationToken: continuationToken)
        .Where(x => x.PartitionKey == nameof(Person));

    var (items, charge, newContinuationToken) =
        await GetAllItemsAsync(query, pageSize);


    context.Response.Headers["X-Continuation-Token"] =
        newContinuationToken;
    
    return new
    {
        count = items.Count,
        requestUnits = charge,
        people = items,
    };
});
```

The above example tries to read a token from a header named `X-Continuation-Token`. This is not required to read the first page. This will then return a token stored in the same header for the response if there is another page.

The header `X-Continuation-Token` was chosen to return the token to a client because, if this was returned in a JSON object the quotes would have to be escaped. This when provided back to Cosmos DB is rejected as an invalid continuation token. See an example of a valid continuation token below.

```json
[
    {
        "token": "+RID:~47VbALuANL8ZAAAAAAAAAA==#RT:1#TRC:25#ISV:2#IEO:65567#QCF:4#FPC:ARkAAAAAAAAALAEAAAAAAAA=",
        "range": {
            "min": "",
            "max": "FF"
        }
    }
]
```

Running the samples and working through the pages of data, for each request the RU charge remains consistent. There is not a large increase in cost as you work further through the pages. This is in large contrast to the _skip_, _take_ approach that costs more the further through the pages you go.

Up to now continuation tokens are looking like the best option. However, there are a few things you cannot do with continuation tokens. The first thing is you can only move forward, you cannot go back a page. The second is that you cannot change the query. Let's say you ordered by age and read the first page. You then got a continuation token back from that request. If you then tried to change the ordering to order by name and passed the same continuation token. Then you would get an error from Cosmos DB. This was something I found the hard way, see  [this GitHub issue](https://github.com/Azure/azure-cosmos-dotnet-v3/issues/2657) for more information.

### Note on query execution in Cosmos DB

There is one part of the two examples I wanted to draw attention to and that is the method that is used in both `GetAllItemsAsync(query, pageSize);`. The implementation of this is below.

```csharp
static async Task<(
    List<Person> items, 
    double charge, 
    string? continuationToken
    )> GetAllItemsAsync(
    IQueryable<Person> query,
    int pageSize)
{
    string? continuationToken = null;
    List<Person> results = new();
    int readItemsCount = 0;
    double charge = 0;
    using FeedIterator<Person> iterator = query.ToFeedIterator();

    while (readItemsCount < pageSize && iterator.HasMoreResults)
    {
        FeedResponse<Person> next = 
            await iterator.ReadNextAsync();

        foreach (Person result in next)
        {
            if (readItemsCount == pageSize)
            {
                break;
            }

            results.Add(result);
            readItemsCount++;
        }

        charge += next.RequestCharge;
        continuationToken = next.ContinuationToken;
    }

    return (results, charge, continuationToken);
}
```

The main points to note in this method are the `while` loop and the `readItemsCount` variable that allows this method to make sure it returns all the available results back from a given paging query.

The reason this is needed is that even though we are paging. The .NET SDK will also perform some paging for larger results sets. The other reason is that when we set the `MaxItemCount` property on the `QueryRequestOptions` object, this is _specifically_ a *maximum* item count. This means in some cases Cosmos DB may return fewer results, even though there are more available. Learn more about this by reading [the docs](https://docs.microsoft.com/en-us/azure/cosmos-db/sql/sql-query-pagination)

## Paging in Cosmos DB using the Cosmos Repository Library

The next set of examples shows how you can achieve paging with the two methods shown using the .NET SDK, using the Cosmos Repository. This library aims to take away a lot of the complexity and boilerplate that is included in the  [.NET SDK sample](https://github.com/mumby0168/cosmos-db-paging/blob/main/CosmosDbSdkPagingSample/Program.cs).

Hopefully, you will see how concise these examples are and furthermore, how the amount of code you are required to write decreases by using this library. See the [full sample here](https://github.com/mumby0168/cosmos-db-paging/blob/main/CosmosRepositoryPagingSample/Program.cs).

> You can find the Azure Cosmos Repository on [GitHub](https://github.com/IEvangelist/azure-cosmos-dotnet-repository)  and on [Nuget].(https://www.nuget.org/packages/IEvangelist.Azure.CosmosRepository) 

### Cosmos Repository Setup

In order to set up the Cosmos Repository, you can use a simple extension provided by the library, which extends the `ServiceCollection` provided by Microsoft. See below.

```csharp
using CosmosRepositoryPagingSample;
using Microsoft.Azure.CosmosRepository;

var builder = WebApplication.CreateBuilder(args);

const string databaseName = "people-database";
const string peopleContainerName = "people-container";
const string partitionKey = "/partitionKey";

builder.Services.AddCosmosRepository(options =>
{
    // In order to set this connection string run
    // dotnet user-secrets set RepositoryOptions:CosmosConnectionString "<your-connection-string>
    // options.CosmosConnectionString = "<taken-from-config>";

    options.DatabaseId = databaseName;
    options.ContainerPerItemType = true;

    options.ContainerBuilder
        .Configure<Person>(containerOptionsBuilder =>
    {
        containerOptionsBuilder
            .WithContainer(peopleContainerName)
            .WithPartitionKey(partitionKey);
    });
});

var app = builder.Build();
```

The only other thing you need to do is ensure that the model you are wanting to use implements the `IItem` interface. There are a set of base classes that are provided by the library that also implements this interface, such as `FulllItem`. This is used in the example of the `Person` object below.

```csharp
class Person : FullItem
{
    public Person(
        string name, 
        int age, 
        string address)
    {
        Id = name;
        Name = name;
        Age = age;
        Address = address;
        PartitionKey = nameof(Person);
    }

    [JsonConstructor]
    private Person(
        string id, 
        string name, 
        int age, 
        string address, 
        string partitionKey)
    {
        Id = id;
        Name = name;
        Age = age;
        Address = address;
        PartitionKey = partitionKey;
    }

    public string Name { get; }
    public int Age { get; }
    public string Address { get; }
    public string PartitionKey { get; }

    protected override string GetPartitionKeyValue() =>
        PartitionKey;
```

The final thing to do is to inject an instance of the `IRepository<Person>` into any consumer and use the extensive set of methods provided. See this _interfaces_ definition [here](https://github.com/IEvangelist/azure-cosmos-dotnet-repository/blob/main/src/Microsoft.Azure.CosmosRepository/DefaultRepository.cs).

### Continuation token paging using the Cosmos Repository

The following endpoint demonstrates how you can use the Cosmos Repository to implement continuation token paging.

```csharp
app.MapGet("/tokenBased", async (
    HttpContext context,
    [FromServices] IRepository<Person> repository,
    [FromQuery] int pageSize) =>
{
    string? continuationToken = 
        context.Request.Headers["X-Continuation-Token"];

    IPage<Person> result = await repository.PageAsync(
        x => x.PartitionKey == nameof(Person),
        pageSize,
        continuationToken);
    
    context.Response.Headers["X-Continuation-Token"] =
        result.Continuation;

    return new
    {
        requestUnits = result.Charge,
        count = result.Total,
        people = result.Items
    };
});
```


### _Skip_, _take_ paging using the Cosmos Repository

The _skip_, _take_ approach using the Azure Cosmos Repository is shown below.

```csharp
app.MapGet("/skipTake", async (
    [FromServices] IRepository<Person> repository,
    [FromQuery] int pageNumber,
    [FromQuery] int pageSize) =>
{

    IPageQueryResult<Person> result = await repository.PageAsync(
            x => x.PartitionKey == nameof(Person),
            pageNumber,
            pageSize);

    return new
    {
        requestUnits = result.Charge,
        count = result.Total,
        pages = result.TotalPages,
        people = result.Items
    };
});
```

The response that reading the 10th page resulted in is also shown below.

```json
{
    "requestUnits": 14.100000000000001,
    "count": 400,
    "pages": 16,
    "people": [
        {
            "name": "Harry Pouros",
            "age": 26,
            "address": "635 Predovic Heights",
            "partitionKey": "Person",
            "etag": "\"0600097c-0000-0d00-0000-61e309330000\"",
            "timeToLive": null,
            "lastUpdatedTimeUtc": "2022-01-15T17:49:39",
            "lastUpdatedTimeRaw": 1642268979,
            "createdTimeUtc": null,
            "id": "Harry Pouros",
            "type": "Person"
        },
        //Excluded the rest of the results for brevity
    ]
}
```

### Why use the Cosmos Repository?

I hope that the code samples above, look concise and succinct that was the aim. Furthermore, the `IPage<TItem>` and `IPageQueryResult<TItem` _interfaces_ offer some extra properties returned back from a paging operation, such as how many items there are in total, how many pages there are, and it also includes the RU charge. These paging implementations also handle all of the query execution bits that were discussed earlier. 

I may be slightly biased, being a passionate maintainer of the Cosmos Repository, but I feel this library offers a super-thin wrapper around the .NET SDK that includes a ton of great features! These features took a lot of time and effort to implement, this is time saved for any consumer of this package in my opinion. I would love anyone reading to try this out and offer any feedback via the [GitHub repository](https://github.com/IEvangelist/azure-cosmos-dotnet-repository).

> It is worth also noting that there has been a lot of work by the EF Core team to integrate Cosmos DB into EF Core. It is certainly also worth considering this an option. See [this blog post](https://devblogs.microsoft.com/dotnet/taking-the-ef-core-azure-cosmos-db-provider-for-a-test-drive/) on some of the new features they have added.

# Final words

Thank you for taking the time to read this and I certainly hope it has helped explain some of the options you have available for paging in Cosmos DB. I am happy to answer any questions feel free to reach out via the comments or tweet me [@billydev5](https://twitter.com/billydev5).
