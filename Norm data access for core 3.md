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

In my opinion **related code should be closer together as possible.**

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

#### 2. Performances. It's faster





Ok, se we do have same testing capabilities and same editor autocomplete features like with class instance models, so what are the benefits?

In my opinion and in my personal experience as I've used in a couple of projects already - it is simply easier and even more elegant.

Does anyone truly enjoys typing all those various DTO classes?

Especially when they are located in another file or even worse on another side of project, so you just have to constantly navigate around, that just hurts code cohesion and adds up to cognitive exertion.

In my opinion it's much easier when related code (model in a form of a tuple and query that generates it) - is close as possible.

And it's bit shorter too. Although, you have to bear in mind that those values are matched by position, not by name, so you have to type data types twice.

As a consequence this approach is **fast as it gets** - even faster then Dapper. [Performance tests](https://github.com/vbilopav/NoOrm.Net#performances) are showing that Dapper averages serialization of one million records in 02.859 seconds and Norm tuples in 02.239 seconds.

There is also one key difference. Dapper will iterate trough results immidatly when called to serialize them (and then you can start building your expression tree using `Linq` to transform the results) -  Norm will not.

Norm will simply create iterator for later iteration, on which you can build your expression tree. And actual iteration will start only when you execute you actually execute it with `foraech` or `ToList` for example. That can save unnecessary data traversals over potentially big result sets.

But, to be completely honest - performance thing does not matter so much. Because, it is noticeable when you start returning millions of rows to a client and if you are doing that, then you are doing something wrong (or just data very intense application).

At the end of the day it may come to personal preferences if you still prefer traditional class instance models, although I suggest you try this approach since I found it to be more convenient.

Luckily, Norm is extendible and one of those extensions is doing just that. So that service method for example above can look like this:

```csharp
public IEnumerable<User> GetUsers() =>
    _connection.Read("select Id, Name, Email from NormUsers").Select<User>();
```

This example uses generic `Select` extension to create an iterator that will serialize to class instances when iteration is executed. Again, you can continue building your `Linq` expression tree to transform results to whatever - and iteration will not commence until you actually execute it with `foreach`, `ToList` or `Count`...

## Asynchronous operations

sdf