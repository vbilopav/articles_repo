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

But when we run this example this is the response that we will see:

> `{}`

Empty JSON.

> That it is because new JSON serializer from .NET Core 3 still doesn't support name tuples.

Not yet anyway. So, in order to have it work - we have to use class instance models:

```csharp
public IAsyncEnumerable<User> GetUsersAsync() =>
    _connection.ReadAsync("select Id, Name, Email from NormUsers").Select<User>();
```

and

```csharp
[HttpGet]
public IAsyncEnumerable<User> Get() => Service.GetUsersAsync();
```

This works as expected.

Now, again, let's add small delay in our iterator expression tree to see does it really streams our response:

```csharp
public IAsyncEnumerable<User> GetUsersAsync() =>
    _connection.ReadAsync("select Id, Name as UserName, Email from NormUsers").Select<User>().SelectAwait(async u =>
    {
        await Task.Delay(100);
        return u;
    });
```

No, not really. Response download will again commence only after all data has been written. So that means if we have 1000 records and each is generated in 100 milliseconds, response download will start only after 1000 * 100 milliseconds.

We'll have to try something else...

Lucky for us, .NET Core 3 is packed with exciting new tech.

## **`Blazor`** pages

`Blazor` is exciting new technology that comes with .NET Core 3. This version uses Blazor Server Side which is hosting model that updates your page asynchronously by utilizing web sockets via SignalR implementation.

So, since it updates web pages asynchronously we might finally get lucky with Blazor. Let's try. 

Add the code block in your page:

```csharp
@code {
    List<User> Users = new List<User>();

    protected override async Task OnInitializedAsync()
    {
        await foreach (var user in Service.GetUsersAsync())
        {
            users.Add(user);
            this.StateHasChanged();
        }
    }
}
```

This page code block defines property `Users` which will hold our results. Every time that property is changed page will be re-rendered to reflect changes in following area:

```csharp
<tbody>
    @foreach (var user in Users)
    {
        <tr>
            <td>@user.Id</td>
            <td>@user.UserName</td>
            <td>@user.Email</td>
        </tr>
    }
</tbody>
```

Also, ye may notice that we have overridden `OnInitializedAsync` protected method:

```csharp
protected override async Task OnInitializedAsync()
{
    await foreach (var user in Service.GetUsersAsync())
    {
        users.Add(user);
        this.StateHasChanged();
    }
}
```

This will be executed after page initialization and we can use that to stream into our reactive property. And since it is in this special initialization event - we have to explicitly inform the page that state has been changed with additional `StateHasChanged` call.

And that's it, this looks good, this should work. And really, when we open this page we can see that table is populated progressively, one by one. So it seems that we've finally achieved asynchronous streaming directly from our database to our page. It only seems so at first.

That is, until we add small delay again to our data service method.

What will happen in that case is real rendering of the page will only start when `OnInitializedAsync` method is completed. State changes are buffered after the execution of that method. So page will first await for that execution and then start rendering asynchronously.

Bummer. That's not exactly what I was hoping to achieve.

But, there is still one option left. Since Blazor is using `WebSockets` - maybe we can utilize them too, to finally have real asynchronous streaming from database to web page.

## **`SignalR`** streaming

`SignalR` is Microsoft implementation of `WebSockets` technology and apparently it does support streaming.

So, first, lets create `SignalR` hub that returns our `IAsyncEnumerable`:

```csharp
public class UsersHub : Hub
{
    public UsersService Service { get; };

    public UsersHub(UsersService service)
    {
        Service = service;
    }

    public IAsyncEnumerable<User> Users() => Service.GetUsersAsync();
}
```

Next, we'll have to build a web page that connects to our streaming hub and with simple client renderer. Entire Razor web page:

```html
@page
@{
    Layout = "_Layout.cshtml";
}

<h1>Razor Page Norm data access example</h1>

<table class="table">
    <thead>
        <tr>
            <th>Id</th>
            <th>Username</th>
            <th>Email</th>
        </tr>
    </thead>
    <tbody id="table-body">
    <!-- content -->
    </tbody>
</table>

<template id="row-template">
    <td>${this.id}</td>
    <td>${this.userName}</td>
    <td>${this.email}</td>
</template>

<script src="~/js/signalr/dist/browser/signalr.min.js"></script>

<script>
(async function () {
    const
        connection = new signalR.HubConnectionBuilder().withUrl("/usersHub").build(),
        tableBody = document.getElementById("table-body"),
        template = document.getElementById("row-template").innerHTML;

    await connection.start();

    connection.stream("Users").subscribe({
        next: item => {
            let tr = document.createElement("tr");
            tr.innerHTML = new Function('return `' + template + '`').call(item);
            tableBody.appendChild(tr);
        }
    });
})();
</script>
```

Yay! With a little help of JavaScript - finally it works as expected:

Data is streamed properly for our database to our web page. It doesn't matter now if we add delay ot our data service, stream will start immidatly. If we add, let's say a one second delay, new row will appear on a web page every second. As soon as database returns data row it sent down the web socket and into our web page where is rendered immidatly.

It's a beautiful thing to watch, it really is, almost brought a tear to my eye ;)

So that is it, I hope I have managed to demonstrate how Norm data access can be useful to anyone developing data application with .NET Core 3 as it is useful to me, and also how to asynchronously stream your data from database to a web page and still being able to keep editor features, models, testability and all of those wonderful things we used to have.

If you have any comment, suggestion or criticisms or anything to add - please let me know in comments down bellow.

