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

## Asynchronous operations

sdf