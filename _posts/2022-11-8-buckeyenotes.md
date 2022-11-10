---
title: Buckeye CTF 2022 - buckeyenotes - Writeup 
date: 2022-11-08 -500
categories: [ctf]
tags: [buckeye2022, writeup, web, sqli]
---

![challenge-description](/assets/buckeyenotes/buckeyenotes-1.png)

Hi! Today we are going to tackle *buckeyenotes* web challenge from 2022 edition of Buckeye CTF. Since the challenge was rated as *begginer*, I wanted to take my time and dive a bit deeper into **what is going on and why**, so that any beginner CTF players reading this can easily follow along.

## Inspecting the website

Let's take a look at the website:

![picture-2](/assets/buckeyenotes/buckeyenotes-2.png)


The first thing you might want to do with any login page is try your luck with some deafult credentials like *admin:password* or *root:password*. However, as you might have guessed, that is not going to work in our case and we are greeted with:
> Invalid username or password

With that out of the way, let's try a standard SQL injection authentication bypass:

> **Username**: test' OR 1=1 --  
> **Password**: x

![picture-4](/assets/buckeyenotes/buckeyenotes-4.png)

Interesting, it looks like we managed to bypass login authentication and were logged in as user *rene*. But what **exactly** happened?

## SQL Injection 

Since the server is running PHP (*index.php* in URL), I am going to assume the database query looks somewhat like this:

```php
$query = "SELECT username FROM users WHERE username ='" . <USERNAME> . "' AND password ='" . <PASSWORD> . "';";
```
Two things to note here:
* *\<USERNAME>* and *\<PASSWORD>* are grabbed from POST request parameters,
* dot (.) operator concatenates strings in PHP. 

And so with 
> **Username**: test' OR 1=1 --  
> **Password**: x

parameters in POST request, the query is:

```php
$query = "SELECT username FROM users WHERE username ='" . "test' OR 1=1 --"  . "' AND password='" . "x" . "';";
```
which executes the following SQL statement:

```sql
SELECT username FROM users WHERE username = 'test' OR 1=1 --'AND password='x'; 
```

The query essentially says: "Give me a user from *users* table that meets either of these requirements:
* user's username is *test*
* 1=1

Also, ignore everything after *--* as it is a comment and should not be executed".

Alright, so the first condition is going to fail as there is no *test* user in the database. However, the second condition is **always** going to be true because, well,  **1 always equals 1**, right? This means that **every** user in the database meets these requirements and so this query returns all users. 

And since I have the benefit of writing this after the files for all challenges were realesed, I can show you exactly what this query would return:

![picture-db](/assets/buckeyenotes/buckeyenotes-db.png) 


 Wait, so is the database going to log us in as **ALL** users at the same time? Not quite. At the time of doing the challenge we obviosuly do not know how the backend works, but it is very likely that it simply takes the **first row in the returned query**. In our case, the first user in *users* table is *rene* and so we get: 

> Logged in as rene . Nothing posted yet :(

The [reason](https://portswigger.net/support/using-sql-injection-to-bypass-authentication) this payload sometimes grants admin privileges is because *admin* user is often the first user in a database.


Now, what would happen if we reversed the payload:

> **Username**: x  
> **Password**: test' OR 1=1 --  

The query would be:

```php
$query = "SELECT username FROM users WHERE username ='" . "x"  . "' AND password='" . "test' OR 1=1 -- " . "';";
```

```sql
SELECT username FROM users WHERE username = 'x' AND password ='test' OR 1=1 --';
```
and would work the same way as before (dumping all users).

![picture-6](/assets/buckeyenotes/buckeyenotes-6.png)

However, it looks like **password** parameter has equal signs filtering - but we can easily work around that with:

> **Username**: x  
> **Password**: test' OR 1<2 --  

as **1 < 2** will also return **true** in SQL query.

![picture-4](/assets/buckeyenotes/buckeyenotes-4.png)

But again, we get logged in as *rene* which does not give us the flag. 


Alright, that was a handful of information, let's finally get to the solution.

## Solution #1 

From the challenge description, we know that user *brutusB3stNut9999* exists somewhere in the database - and there is a good chance this user has the flag. That makes it very convenient as we can simply use:

> **Username**:  brutusB3stNut9999' --  
> **Password**: x 

as payload which results in the following query:

```sql
SELECT username FROM users WHERE username = 'brutusB3stNut9999' --'AND password='x';
```

And sure enough, we get the flag:

![picture-6](/assets/buckeyenotes/buckeyenotes-8.png)

## Solution #2

The second option is to extend the query and ask for the username *brutusB3stNut9999*. 

> Username: x  
> Password: test' OR 1<2 AND username LIKE 'brutusB3stNut9999' -- 

```sql
SELECT username FROM users WHERE username ='x' AND password='test' OR 1<2 AND username LIKE 'brutusB3stNut9999' --';
```

The *LIKE* keyword is used for pattern match comparison and allows us to bypass equal signs filtering. Normally, it could be used like this: ```AND username LIKE 'brutus%' -- ``` and it would return all usernames that matched the pattern, so for example: *brutusB3stNut123*, *brutusW0rstNut9999*,  *brutusB3stNut9999*. In this case, we used it to match exactly the username we are looking for.


As you can see, the first pair of requirements  ```username ='x' AND password='test'``` is going to fail. The second pair  ```1<2 AND username LIKE 'brutusB3stNut9999'``` , however, is true and so the database logs us in as *brutusB3stNut9999* and we get the flag, same as earlier.

![picture-6](/assets/buckeyenotes/buckeyenotes-8.png)

Note that in this case we could have also reversed the payload and it would still work:

> Username: test' OR 1<2 AND username LIKE 'brutusB3stNut9999' --  
> Password:  x



```sql
SELECT username FROM users WHERE username ='test' OR 1<2 AND username LIKE 'brutusB3stNut9999' --' AND password='x';
```



## Conclusion

I know this was a rather long article for such a simple challenge, but I hope it could help begginer CTF players better understand how SQL Injection works. 

Also, somewhere in the article I said:
> *At the time of doing the challenge we obviosuly do not know how the backend works, but it is very likely that it simply takes the first row in the returned query.*

If you are curious about how exactly did the backend (and SQL queries in particular) worked in this challenge, go can take a look at [Buckeye CTF 2022 official repo](https://github.com/cscosu/buckeyectf-2022-public/tree/master/web/buckeyenotes).

