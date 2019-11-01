# What is Norm for .NET

**`Norm` is a data access library for .NET Core 3 (or .NET Standard 2.1).**

Previously called `NoORM` to give emphases to fact that this is Not Yet Another ORM (although I have added O/R mapping extension recently) - now it's just shortened to `Norm`.

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

Notice anything unusual?

Well, **there is no model** here that is returned from our service. No model at all.

For such scenarios we're all used to (self included) of having models. Models are great, they give us benefits of editor autocompletion, they gives testability and so on.

But `Norm` doesn't return model by default - `Norm` returns tuples because that's what databases do return - data tuples.

However, in this example we use c# concept of [named tuples](https://docs.microsoft.com/en-us/dotnet/csharp/tuples#named-and-unnamed-tuples) to give names for our values, so **we still have same benefits** as data model based on class instance as we used to.

In a sense named tuples act like a class instance data model.

We could go even step further and use unnamed tuples and give them names when we use them in a page, by using [tuple deconstruction](https://docs.microsoft.com/en-us/dotnet/csharp/deconstruct#deconstructing-a-tuple) feature like this:

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

Ok, se we do have same testing capabilities and same editor autocomplete features like with class instance models, so what are the benefits?

In my opinion and in my personal experience as I've used in a couple of projects already - it is simply easier and even more elegant.

Does anyone truly enjoys typing all those various DTO classes? Especially when they are located in another file or even worse on another side of project, so you just have to constantly navigate around, that just hurts code cohesion and adds up to cognitive exertion.

And it's bit shorter too