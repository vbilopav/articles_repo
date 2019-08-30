# How to write super-uber-mega secure, sql-injection bullet-proof PostgreSQL queries

SQL injection is big problem in modern software development.

Even when you have a team trained familiar with best cybersecurity practices, there is always a danger that something (by accident or by flaw in software) might slip trough.

I'll describe here how to make your PostgreSQL queries safe as they can be of any potential SQL injection vulnerabilities.

What I'm going to show is nothing new. It is actually organizational pattern that people used for maximum security, mostly in army and any organization where such informational security measures are needed.

It is called **"Least Privilege Security"**.

Imagine you are sending a soldier to a secret mission behind enemy lines. What you want to do is give him a bare minimum of information needed to complete the mission. So, if he falls in enemy hands - he can't reveal more than he know. Principle is also called **"need-to-know basis".**

Same principle can be applied to any modern database. Let's see how we can achieve that in PostgreSQL.

Let's say we have following query that returns value from table values:

```sql
select "value" from "values" where id = @id;
```

Now, if you input for your @id variable is left unsanitized it might cause SQL injection. Attacker might gain access to your database and steal or delete your data.

So what can we do?

To secure our queries we will need at least two distinguished database users:

1. One with higher privileges, that have create and select grants for this example. It doesn't have to be admin or super user per se, for this example it only needs to be able to create new function on schema, select "values" table and give some grants to other user. We can call it `ddl_admin`.

2. And the other with no privileges and no grants at all. It only may perform database login and that is it. We can call it `app_user` and that is user that your application will use for now on. You can create such minimum privilege (or least privilege) user with following statement:

```sql
create role app_user with
		login
		nosuperuser
		nocreatedb
		nocreaterole
		noinherit
		noreplication
		connection limit -1
		password 'app_user_password';
```

User `ddl_admin` will be user under whose context we will run database migrations and updates.

So we will use that to wrap our query into database function, with few other things, I'll explain, like this:

```sql
create function select_value(_id text) returns text as
$$
begin
	return (
		select "value" 
		from "values" 
		where id = _id
	);
end
$$ language plpgsql security definer;	
revoke all on function select_value(text) from public;
grant execute on function select_value(text) to app_user;
```

So, we wrapped our query into function.

We also said with statement `security definer` that anyone who runs this function will do so as user who actually created it (the definer). And that is, in this case "ddl_admin" and that user have the grant to select values table.

Next statement will revoke all grants on that function for public. When function is created on PostgreSQL by default everyone has execute grants. So we must first revoke that.

At this point, only `ddl_admin` can execute this function. So we must change this and grant execute to `app_user`. This means that now, `app_user` has only one single grant and that is to execute that function:

```sql
select from select_value('1');
```

Will return expected results if executed under `app_user` user context.

Note that `app_user` can do only that and nothing else. So if we attempt to perform SQL injection again, like this, for example:

```sql
select select_value('');select * from some_other_table;
```

we will get as result:

```
ERROR:  permission denied 
```

And that is it, **SQL injection is now complete impossibility on your system.**

Does that means that we are out of the woods yet. Well, no, not really. There are couple of things to have in mind:

1) Always remember to have application with **minimum grants.** For example, if attacker knows that function uses some other function or operator, attacker may, before calling your function change search_path to another schema, create new function or operator there and let your function be executed normally. At this scenario, that new function will be executed with elevated privileges. And that is, of course complete impossibility if your application user doesn't have any creation grants.

2) Some functions may use something called dynamic SQL. Dynamic SQL is string that contains SQL statements and it is executed by calling EXECUTE function. Now, if that dynamic query uses some parameters, of course, SQL injection could happen there. What to do about it. Honestly, I'd just avoid dynamic SQL all together if it is possible.
