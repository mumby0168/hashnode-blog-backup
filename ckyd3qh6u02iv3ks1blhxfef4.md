---
title: "Fluent .NET Apis - Guiding A Developer"
seoTitle: "Fluent .NET Apis - Guiding A Developer"
seoDescription: "Sometimes when creating an object or performing a task via an API the creator of that API may want to ensure certain things happen sometimes in a specific o"
datePublished: Thu Jan 13 2022 15:03:35 GMT+0000 (Coordinated Universal Time)
cuid: ckyd3qh6u02iv3ks1blhxfef4
slug: fluent-dotnet-apis-guiding-a-developer
cover: https://cdn.hashnode.com/res/hashnode/image/unsplash/b2Cf5xAsmLQ/upload/v1642086192212/9cT4mfzWD.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1642086157589/I-N4vAd3f.png
tags: api, dotnet, dotnetcore

---

# Fluent Apis - Guiding A Developer

Sometimes when creating an object or performing a task via an API the creator of that API may want to ensure certain things happen sometimes in a specific order. It may also be the case that some things are required to happen and some things are not. A fluent API is a great way to guide a developer into a direction and also provides a great experience for them while being a client of your API.

There is a high likelihood that if you are reading this and have heard of the term *fluent API* or not that you will have used one before. It is highly likely that you will have made an API or seen an API made in ASPNET Core. If you have then you will have seen the `Startup` class. This class has two methods as shown below. Note that this example, shown below does not make separate calls to `services` or `app` it chains all of these calls together.

```c#
public class Startup
{
    public void ConfigureServices(IServiceCollection services) =>
        services.AddCosmosRepository(o =>
            {
                o.DatabaseId = "dogs-db";
                o.ContainerId = "dogs";
            })
            .AddSwaggerGen()
            .AddMvc();

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env) =>
        app.UseSwagger()
            .UseSwaggerUI()
            .UseRouting()
            .UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
                endpoints.MapGet("/", async context =>
                {
                    await context.Response.WriteAsync("Dog API");
                });
            });
}
```

### What makes an API fluent?

In the example of the `Startup` class shown above, we can see that every call to register some `services` or to configure some middleware on the `app` return an instance of the interface back from the method call. `IServiceCollection` and `IApplicationBuilder` respectively. The example shown above is one of the more flexible fluent APIs as it just returns an instance of the class over & over it does not guide a user down a specific path.

## Guiding a developer
In order to guide a developer down a specific path when creating a fluent API, interfaces are used to restrict the next method that the user has access to when chaining the methods calls together. This is best shown via an example. This example will show an `AccountBuilder`, this has a few requirements.

1. An account must have an email address.
2. An account must have a password.
3. An account must verify the password used.
4. An account must have a password with a length of at least 6 characters.
5. An account must have a password with at least one of the special characters `@$`.
6. An account can optionally have a username.
7. An account can optionally have many roles.

In order to meet these requirements, an object has been defined for an account that exposes all of these properties to meet the requirements above. See below.

```c#
public class Account
{ 
    public string Id { get; set; } = Guid.NewGuid().ToString();

    public string Email { get; set; } = string.Empty;

    public string Password { get; set; } = string.Empty;

    public string? Username { get; set; }

    public List<string> Roles { get; set; } = new();
}
```

This class is all well and good but defines a public constructor. This means a client to this class could create it has not met any of the requirements listed above. What would be better is if we made this constructor internal and made the client create an account via a fluent API named `AccountBuilder`. This is going to make a user first, provide their email address, then provide a password, then verify that password before optionally providing a username and some roles. In order to let a user start the process, we will define a factory that returns an interface that only allows the user to initially provide their email address see the example below.

```c#
public interface IAccountBuilderWithEmail
{
    IAccountBuilderWithPassword WithEmailAddress(string email);
}

public static class AccountFactory
{
    public static IAccountBuilderWithEmail AccountBuilder() => new AccountBuilder();
}
```

The example above show's how the factory initially only lets the user of the api call the method `WithEmailAddress(...)`. Until they have done this they do not get access to the interface named `IAccountBuilderWithPassword` which will then let them provide a password. See below.

```c#
public interface IAccountBuilderWithPassword
{
    IAccountBuilderWithVerifiedPassword WithPassword(string password);
}
```

This method then lets them verify this password via the interface `IAccountBuilderWithVerifiedPassword` but again nothing more. What this example ends up with is an interface per method that must be called and once completed, the method returns the next method that must be called. This is in essence guiding a developer through the process of creating an account. See all of the interfaces below to see how this starts to guide the user up until the optional part of the process is defined by the interface `IAccountBuilder`.

```c#
public interface IAccountBuilderWithEmail
{
    IAccountBuilderWithPassword WithEmailAddress(string email);
}

public interface IAccountBuilderWithPassword
{
    IAccountBuilderWithVerifiedPassword WithPassword(string password);
}

public interface IAccountBuilderWithVerifiedPassword
{
    IAccountBuilder VerifyPassword(string password);
}

public interface IAccountBuilder
{
    IAccountBuilder WithUsername(string username);

    IAccountBuilder WithRole(string role);

    IAccountBuilder WithRoles(params string[] roles);

    IAccountBuilder WithRoles(IEnumerable<string> roles);

    Account Build();
}
```

See how this is starting to come together? I hope so. The final interface that is returned once all the required methods have been called is `IAccountBuilder`. This then optionally allows the user to define roles & a user name before getting access to the `Account` object by calling `Build()` which completes the process of building an account.

The neat part about this is that all of the implementations for our fluent API can live inside a single class called `AccountBuilder` even though there are many interfaces guiding a client through this process. See the implementation below.

```c#
class AccountBuilder : 
        IAccountBuilderWithEmail, 
        IAccountBuilderWithPassword, 
        IAccountBuilderWithVerifiedPassword, 
        IAccountBuilder
    {
        readonly Account _account = new();

        public IAccountBuilderWithPassword WithEmailAddress(string email)
        {
            _account.Email = email;
            return this;
        }

        public IAccountBuilderWithVerifiedPassword WithPassword(string password)
        {
            if (password.Length < 6)
                throw new InvalidOperationException("A password must be at least 6 characters.");

            if (password.Contains('@') is false && password.Contains('$') is false)
                throw new InvalidOperationException("A password must contain one of the special characters '@$'.");

            _account.Password = password;
            return this;
        }

        public IAccountBuilder VerifyPassword(string password)
        {
            if (_account.Password != password)
                throw new InvalidOperationException("The passwords provided do not match.");

            return this;
        }

        public IAccountBuilder WithUsername(string username)
        {
            _account.Username = username;
            return this;
        }

        public IAccountBuilder WithRole(string role)
        {
            _account.Roles.Add(role);
            return this;
        }

        public IAccountBuilder WithRoles(params string[] roles)
        {
            _account.Roles.AddRange(roles);
            return this;
        }

        public IAccountBuilder WithRoles(IEnumerable<string> roles)
        {
            _account.Roles.AddRange(roles);
            return this;
        }

        public Account Build() => _account;
    }
```

Notice how this class implements all of the interfaces and after each method call it just returns `this` or itself back to the client to carry on building that account.

## Putting it all together

The sample for this post can be found [here](https://github.com/mumby0168/blog-samples/tree/main/FluentAccountApi.Sample) and contains a console application which builds an `Account` and generates some JSON for that `Account`. See below.

### Console Application

```c#
using System;
using System.Text.Json;
using FluentAccountApi.Sample;

Console.WriteLine("Starting the account builder demo.");

Account account = AccountFactory
    .AccountBuilder()
    .WithEmailAddress("joe.bloggs@gmail.com")
    .WithPassword("Test123@")
    .VerifyPassword("Test123@")
    .WithRoles("admin", "super")
    .WithUsername("joe123")
    .Build();

Console.WriteLine("Built account ->");
Console.WriteLine(JsonSerializer.Serialize(account, new JsonSerializerOptions{WriteIndented = true}));
```

### Output
```
Starting the account builder demo.
Built account ->
{
  "Id": "dc2cc20f-153c-4371-a053-caa071012773",
  "Email": "joe.bloggs@gmail.com",
  "Password": "Test123@",
  "Username": "joe123",
  "Roles": [
    "admin",
    "super"
  ]
}
```

## Final Words
Fluent APIs can really help a developer through the process of building an object from your library. They can be used in many places and a lot of developers really like using them. Please try out the example provided and get to grips with how it works. Hopefully, you will start to notice fluent APIs in the .NET ecosystem or maybe even create your own!

Find the source code here: https://github.com/mumby0168/blog-samples/tree/main/FluentAccountApi.Sample
