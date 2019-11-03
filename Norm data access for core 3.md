# What is Norm for .NET

**`Norm` is a data access library for .NET Core 3 (or .NET Standard 2.1).**

Previously called `NoORM` to give emphases to fact that this is Not Yet Another ORM (although I have added O/R mapping extension recently) - now it's just shortened to **`Norm`**.

I've built it for my needs as I do frequent work on data intense applications, and it is fully tested with PostgreSQL and Microsoft SQL Server databases.

You can find official repository [here](https://github.com/vbilopav/NoOrm.Net).

Now I'm going to try to demonstrate how it can be very useful to anyone building .NET Core 3 data intense applications.

## Example

Example uses very simple example data model with just three table:

- `NormUsers` - Table of test users with name and email
- `NormRoles` - Table of test roles with role name
- `NormUserRoles` - Junction table so we can have many-to-many relationship between users and roles

![Data model](https://raw.githubusercontent.com/vbilopav/articles_repo/master/norm-model.jpg)

Data definition script is [here](https://github.com/vbilopav/NormExamples/blob/master/Data/Migrations/20191023132952_CreateSchema.cs).

Also, example provides migration that inserts 4 roles and generates 1000 initial users with random generated names and random generates emails. Script is located [here](https://github.com/vbilopav/NormExamples/blob/master/Data/Migrations/20191023133504_InsertData.cs), but naturally, anyone can clone or download and tweak it to generate more users for testing purposes (go for a million).

Now, let's get to examples.

## Read users and show them on a web page (using razor page)

Assuming that we have [configured](https://github.com/vbilopav/NormExamples/blob/master/Startup.cs#L38) a service that hold our database [connection](https://github.com/vbilopav/NormExamples/blob/master/Data/UsersService.cs#L25), we can add very simple `GetUsers` method like this:

```csharp
public IEnumerable<(int id, string userName, string email)> GetUsers() =>
    _connection.Read<int, string, string>("select Id, Name, Email from NormUsers");
```

Now we can show our data in a table body element of our page:

```csharp
<tbody>
    @foreach (var user in Model.Service.GetUsers())
    {
        <tr>
            <td>@user.id</td>
            <td>@user.userName</td>
            <td>@user.email</td>
        </tr>
    }
</tbody>
```

### Notice anything unusual?

> Well, **there is no instance model** here that is returned from our service.
> **No instance model at all.**

For such scenarios - we're all used to (self included) of having **class instance models.**

Those models are great - they give us benefits of **editor autocompletion**, enhance our testability and readability and so on.

But **`Norm`** doesn't return that kind of model by default.
> **`Norm` returns tuples by default**, because that's what databases do return - data tuples.

However, in this example we use c# concept of **[named tuples](https://docs.microsoft.com/en-us/dotnet/csharp/tuples#named-and-unnamed-tuples)** - to give names for our values.

> **We still have same benefits** as data model based on class instances as we used to. Editor autocompletion, intellisense, testability - everything is still there,

> In a sense - named tuples act like a class instance data model.

We could go even step further and **use unnamed tuples** and give them names when we use them in a page, by using **[tuple deconstruction](https://docs.microsoft.com/en-us/dotnet/csharp/deconstruct#deconstructing-a-tuple)** feature like this:

```csharp
<tbody>
    @foreach (var (id, userName, email) in Model.Service.GetUsers())
    {
        <tr>
            <td>@id</td>
            <td>@userName</td>
            <td>@email</td>
        </tr>
    }
</tbody>
```

In that case naming our tuples wouldn't even be necessary (although we can if want):

```csharp
public IEnumerable<(int, string, string)> GetUsers() =>
    _connection.Read<int, string, string>("select Id, Name, Email from NormUsers");
```

But then we still have to type data types twice.

Of course, unless we expose our connection to the page (which is **gross anti pattern** since it compromises testability and general maintenance, but it may be good enough for something really quick):

```csharp
<tbody>
    @foreach (var (id, userName, email) in Model.Connection.Read<int, string, string>("select Id, Name, Email from NormUsers"))
    {
        <tr>
            <td>@id</td>
            <td>@userName</td>
            <td>@email</td>
        </tr>
    }
</tbody>
```

### Benefits

What are they?

#### 1. I find to be much more convenient, easier and even faster to develop

For me a s developer higher code cohesion counts. I don't want to navigate somewhere else, into another class, another file, or even another project in same cases - to work on a result of that query from that particular method.

In my opinion - **related code should be closer together as possible.**

This:

```csharp
public IEnumerable<(int id, string userName, string email)> GetUsers() =>
    _connection.Read<int, string, string>("select Id, Name, Email from NormUsers");
```

> - allows you to **see your model which is returned immidatly.** Not just name of your model. The actual model.

However, it may come to personal preferences, although I suggest you try this approach.

In that case, **`Norm`** has **extendible architecture** and class instance mapper extension is included by default. It's generic `Select` extension and you can use it like this:

```csharp
public IEnumerable<User> GetUsers() =>
    _connection.Read("select Id, Name, Email from NormUsers").Select<User>();
```

> **`Norm`** works simply by creating **iterator for later use** over your query results.

That means that `Read` method is executed immidatly in a millisecond, regardless of your query. You can start building your expression trees by using `Linq` extensions (or built-in Norm extension such as this generic `Select` in example above).

> Actual database operation and actual reading and serialization will not commence until you call actual iteration methods such as `foreach`, `ToList` or `Count` for example.

Example:

```csharp
public IEnumerable<(int id, string userName, IEnumerable<string> roles)> GetUsersAndRoles() =>
    _connection.Read<int, string, string>(@"
                select u.Id, u.Name, r.Name as Role
                from NormUsers u
                    left outer join NormUserRoles ur on u.Id = ur.UserId
                    left outer join NormRoles r on ur.RoleId = r.Id
                ")
        .GroupBy(u =>
        {
            var (id, userName, _) = u;
            return (id, userName);
        }).Select(g => (
            g.Key.id,
            g.Key.userName,
            g.Select(r =>
            {
                var (_, _, role) = r;
                return role;
            })
        ));
```

this method transform users and their roles to composite structure where each user have its own enumerator for roles.

#### 2. Performances. It's a bit faster

Yes it's bit faster, even from [`Dapper`](https://github.com/StackExchange/Dapper), but only when you use tuples approach described above.

That is because tuples are mapped by position, not by name, name is irrelevant.

[Performance tests](https://github.com/vbilopav/NoOrm.Net#performances) are showing that Dapper averages serialization of one million records in `02.859` seconds and Norm generates tuples in `02.239` seconds.

Difference may be even higher since Norm can be used to avoid unneccessary iterations by building iterator for query results.

For example, Dapper will typically iterate once to generate results and then typically you'll have to to another iteration to do something with those results, to generate json response for example. For contrast, Norm will create iterator, then you can build your expression tree and ideally **execute iteration only once.**

> It's just smart way to avoid unneccessary iterations in your program.

But, to be completely honest - that whole performance thing may  not matter that much.

Because, it is noticeable when you start returning millions and millions of rows from database to a your database client.

> And if you are doing that - returning millions and millions of rows from database - then you are doing something wrong (or just data very intense application).

## Asynchronous operations and asynchronous streaming

For asynchronous operations Norm doesn't return `Task` object like some might expect. Instead, it returns [**`IAsyncEnumerable`**](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1?view=dotnet-plat-ext-3.0)

From docs:
> `IAsyncEnumerable` exposes an enumerator that provides asynchronous iteration over values of a specified type.

`IAsyncEnumerable` is new type for .NET Core that got many people excited, self included. It finally allows real asynchronous streaming.

> The goal here is to write the values to your output response stream **as they appear on your database connection** - while still preserving features we're used to have described above (intellisense, autocomplete, models, testing, etc).

Let's modify our service method to use asynchronous version:

```csharp
public IAsyncEnumerable<(int id, string userName, string email)> GetUsersAsync() =>
    _connection.ReadAsync<int, string, string>("select Id, Name, Email from NormUsers");
```

There is no `async` and `await` keywords any more, because we're not returning `Task` object, so we don't need them.

Now, we can put this new `async foreach` feature to good use - and render our page like this:

```csharp
<tbody>
    @await foreach (var user in Model.Service.GetUsersAsync())
    {
        <tr>
            <td>@user.id</td>
            <td>@user.userName</td>
            <td>@user.email</td>
        </tr>
    }
</tbody>
```

What happens here is that database values are written to our page as they appear on our database connection (and we still have intellisense, autocomplete and all that).

Let's compare this to what we have used to do before .NET Core 3 and `IAsyncEnumerable` type, for example, we could also have generated our page like this:

```csharp
<tbody>
    @foreach (var user in await Model.Service.GetUsersAsync().ToListAsync())
    {
        <tr>
            <td>@user.id</td>
            <td>@user.userName</td>
            <td>@user.email</td>
        </tr>
    }
</tbody>
```

This is something we used to do regularly in pre .NET Core 3 era (although, there wasn't `ToListAsync` method, but I'll get to that in a second).

What it does it actually it waits, although asynchronously, but it still waits first, and only when database connection has returned all values - it writes them down to our page.

> So, asynchronous streaming and `IAsyncEnumerable` type is quite an improvement.

In example above we used non-standard extension called `ToListAsync`. This extension method is part of the new library for .NET Core 3, that just got out of preview that implements standard `Linq` extensions for asynchronous operations over `IAsyncEnumerable` type.

It is called [`System.Linq.Async`](https://www.nuget.org/packages/System.Linq.Async), developed and maintained by .NET Foundation and Contributors and it is referenced by Norm package.

This extension method in particular `ToListAsync` - extends `IAsyncEnumerable` to return a asynchronous `Task` that returns a list object generated by our enumerator.

So, this first version with `async foreach` should by much more efficient because it is asynchronous streaming, right. But we run the page with that implementation we may notice something interesting:

> Page download doesn't start until entire page has been generated.

We can see that clearly if we add small delay to our service method:

```csharp
public IAsyncEnumerable<(int id, string userName, string email)> GetUsersAsync() =>
    _connection.ReadAsync<int, string, string>("select Id, Name, Email from NormUsers").SelectAwait(async u =>
    {
        await Task.Delay(100);
        return u;
    });
```

In this example `SelectAwait` extension method (part of `System.Linq.Async` library, can create projection from async task) - will add expression to expression tree that adds small delay which is executed when we execute our `async foreach` iteration on a page.

So, with that delay, we can see clearly that page is downloaded only when stream is finished.

Not exactly what I was hoping for, let's try something else:

## REST API Controller

If not with Razor Web Page, let's if we can asynchronously stream content to a web page by using REST API and client render.

Luckily, .NET Core 3 REST API Controllers do support `IAsyncEnumerable` type of the box, so that means that we can just return `IAsyncEnumerable` out of the controller and framework will recognize it and serialize it properly.

```csharp
[HttpGet]
public IAsyncEnumerable<(int id, string userName, string email)> Get() => Service.GetUsersAsync();
```

Note that in this case, again, `async` and `await` aren't necessary any more.

But when we run