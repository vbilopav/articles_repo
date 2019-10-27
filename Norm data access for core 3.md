# What is Norm for .NET?

`Norm` is a data access library for .NET Core 3 (or .NET Standard 2.1). Previously called `NoORM` those emphases that this is not yet another ORM (although I have added O/R mapping extension recently), now it's just shortened to `Norm`.

I've built it for my needs as I do frequent work with data intense applications, and it is fully tested with PostgreSQL and Microsoft SQL Server databases.

You can find official repository [here](https://github.com/vbilopav/NoOrm.Net).

Now I'm going to show how it can be useful to you, if you're a .NET developer working with databases, and we'll do some cool stuff like async streaming directly from database to web page.

# Example

```SQL
use NormExample
GO
create table dbo.NormUsers (Id int not null primary key, Name nvarchar(64) not null, Email nvarchar(64) not null)
GO
create table dbo.NormRoles (Id int not null primary key, Name nvarchar(256) not null)
GO
create table dbo.NormUserRoles
(
    UserId int not null,
    RoleId int not null,
    primary key (UserId, RoleId),
    foreign key (UserId) references dbo.NormUsers (Id),
    foreign key (RoleId) references dbo.NormRoles (Id)
)
GO
```