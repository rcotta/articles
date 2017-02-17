# MongoDB permissinos setup

So now it's time to take that MongoDB based project seriously, and you want to secure your database. But you noticed that things are a little bit different from MySQL's `grant all on ...` and `flush privileges`.

The user you created does not have permission on what you believe to be his own database, your admin cannot list collections and what's `--authenticationDatabase` used for.

Just relax and come with me.

## First things first: enable authentication and setup an admin user

Find your mongod.conf file (`/etc/mongod.conf` on Ubuntu; on Windows you may have to create a `mongod.cfg` file as explained [on this link](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-windows/#configure-a-windows-service-for-mongodb-community-edition), step #3) and add the following lines to the file to enable authentication:
````
# setting authentication enabled
security:
  authorization: enabled
````

Dont forget to logon on MongoDB and then create an user with administrative rights on admin database (follow the instructions for now; I'll try to clarify what's going on later):
````
use admin;
db.createUser(
  {
    user: "root",
    pwd: "password",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
  }
)
````

Restart your MongoDB instance and, hopefully, you will no be able to login with no credentials.

## How authentication works on MongoDB

MongoDB authentication works with users created on the target database and permissions based on roles.

The commands above created an user `root` on the database admin (that's the purpose of the `use admin` before the `db.createUser`) with privileges to admin any database no this server. Note that this user won't be able to list a simple collection on any database, but this user will be able to create another user and grant permissions.

What we'll do now is to create an user with read only privileges and another one with read/write privileges on our application's database.

## Logging in with credentials

Before we continue, I'll have to explain how to login to the now secured database. It's not that hard, use:
````
mongo <host>:<port>/<database> -u <username> -p --authenticationDatabase <password>
````

If we use the sample data from the beginning of this article, we'll use something like (-p is for asking for your password; you may want to use `-p <password>`, but entering your password in command line this way is not a good idea):
````
mongo 127.0.0.1:27017/admin -u root -p --authenticationDatabase admin
````

Once your're in, it's time to create additional users for your application database that we'll name `app_db`:
````
use app_db
db.createUser(
  {
    user: "appdb_user",
    pwd: "password",
    roles: [ { role: "readWrite", db: "app_db" } ]
  }
)
````

You should see a "Successfully added user" message, and you'll be able to test this new user by logging in like:
````
mongo 127.0.0.1:27017/app_db -u appdb_user -p
````

And then creating a new collection and listing its contents:
````
db.new_collection.save({'key':'value'});
db.new_collection.find().pretty()
````

Finally, let's create a read-only user for this very same database (using the root login previously detailed):
````
use app_db
db.createUser(
  {
    user: "appdb_user_readonly",
    pwd: "password",
    roles: [ { role: "read", db: "app_db" } ]
  }
)
````

And check if we're able to write using the read only user:
````
mongo 127.0.0.1:27017/app_db -u appdb_user_readonly -p
````

After the login these commands should work:
````
show collections
db.new_collection.find().pretty()
````

And this one should fail:
````
show collections
db.new_collection.save({"some":"data"})
````

## Side notes

Take a look at MongoDB's [security checklist](https://docs.mongodb.com/manual/administration/security-checklist/) before publishing your system on a production environment.

--
Rafael Cotta
