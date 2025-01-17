Our users' table currently has two rows of data, a `tyler clark` and a `debbie jones` that currently share the same user handle and `create_date`.

```sql 
select * from Users; 
  create_date |             user_handle              | last_name | first _name 
--------------+--------------------------------------+-----------+-------------
  2018-06-06  | a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11 | clark     | tyler  
  2018-06-06  | a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11 | jones     | debbie  
(2 rows )
```

We want our `user_handle` to be unique, which is why when we created this table, we gave this column a `uuid` type. Let's update my `user_handle` to be a different value. Postgres has a handy `create extension` we can bring in that will automatically generate a random `uuid` for us. 

```sql 
create extension "uuid-ossp";
CREATE EXTENSION
```

I'll post links in the description on how to work with `uuid`'s in other databases. Now with this extension created, we can update our `Users` table with a new `user_handle`. 
 
```sql 
update Users set user_handle = uuid_generate_v4();
UPDATE 2
```

When working with the update command, there are three pieces of data needed. First is the name of the table and column we're updating. Here, we provide the table name and column name separated by this `set` word, `update Users set user_handle`. Second is the new value that we want to update the row with, `uuid_generate_v4();`. We'll do that by using the equal sign,`=`. Finally, which row to update? You might have noticed that we actually ended up updating both rows of data when we just wanted to update my `user_handle`.

```sql 
select * from Users; 
  create_date |             user_handle              | last_name | first _name 
--------------+--------------------------------------+-----------+-------------
  2018-06-06  | ddda5534-f29b-4583-83c0-21b84e0c9786 | clark     | tyler  
  2018-06-06  | 6ab3b2d2-8e02-45e4-890c-61a67cd43f31 | jones     | debbie  
(2 rows )
```

While it did accomplish our goal of having unique user handles, if we're not careful, we could have shot ourselves in the foot by accidentally losing critical data. When working with SQL, commands like `select`, `update`, `delete`, these are looping commands, meaning unless a condition is passed it will loop over each row of data. Ideally, what we should have done is add a `where` clause on our command that only updated the `user_handle` for the row where the `last_name = 'clark';`. 

```sql 
update Users set user_handle = uuid_generate_v4() where last_name = 'clark';
UPDATE 1
```

We'll get more into filtering and the `where` clause in another video. For now, all you needed to understand is that when updating, unless a condition is provided like we did the `last_name = 'clark'`, it will apply to every single row in our table. 

```sql 
select * from Users; 
  create_date |             user_handle              | last_name | first _name 
--------------+--------------------------------------+-----------+-------------
  2018-01-06  | 6ab3b2d2-8e02-890c-bb6d-61a67cd43f31 | jones     | debbie  
  2018-06-06  | fh37294e-f839-f783-9ffe-7fh3os789822 | clark     | tyler  
(2 rows )
```

Finally, we don't have to update one column at a time. We can update more than one column by simply comma separating them. 

```sql 
update Users set user_handle = uuid_generate_v4(), first_name = 'danny'  where last_name = 'clark';
UPDATE 1
```

By adding `first_name = 'danny'`, I'm updating both my `user_handle` and my `first_name`.

```sql 
select * from Users; 
  create_date |             user_handle              | last_name | first _name 
--------------+--------------------------------------+-----------+-------------
  2018-01-06  | 6ab3b2d2-8e02-890c-bb6d-61a67cd43f31 | jones     | debbie  
  2018-06-06  | 2839f831-f82c-faj3-aof3-fj28ddks39ek | clark     | danny  
(2 rows )
```