---
layout: post
title: PostgreSQL Triggers and Notify 
description: Introduction to LISTEN/NOTIFY in PostgreSQL and a look into potential pitfalls. 
summary: How to set up a TRIGGER in a psql database and listen to notifications using a simple Python console application. 
comments: true
tags: [database, postgresql, python]
---

The PostgreSQL protocol offers a streaming protocol that implements asynchronous messages and notifications. This means that your database server can send out messages at any time. It sounds really nice, but the Notify feature actually comes with pitfalls. The main thing to note is that this feature is not a proper message queue in the sense that there is no mechanism to keep track of what messages have been received. I did compile a list of situations that PSQL Listen/Notify falls short at the end, but let’s go through it for a bit before starting to go through the issues with it.

### Quick Demo
#### Setting Up Our DB Triggers
In order to set up a system where you can subscribe to notifications from your database, you’ll first need some sort of function that notifies specified channels. After you create that function, you’ll need to create a trigger that calls that function to notify channels. 

Imagine a scenario where your neighbour is an obsessive pet lover who’s keeping a database of all the pets in the building. You can choose to subscribe to their database as a dog lover or a cat lover. Then, whenever they insert a new dog to their database, the dog lover channel will get a notification.

Let’s blackbox the notify function for now and call it notify_dog_lovers() and instead look at the trigger. 

```sql
-- Create the trigger
CREATE TRIGGER new_dog_alert
    AFTER INSERT ON public.apt_pets
    FOR EACH ROW EXECUTE FUNCTION notify_dog_lovers();
```

So, this trigger simply creates a trigger called new_dog_alert which executes a function called notify_dog_lovers() whenever an insert happens on apt_pets .

Now, let’s see what `new_dog_alert()` would look like. 

```sql
-- Create the trigger function
CREATE OR REPLACE FUNCTION new_dog_alert() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.type LIKE 'dog' THEN
        -- Notify the channel 'dog_lovers'
        PERFORM pg_notify('dog_lovers', json_build_object(
            'operation', TG_OP,
            'name', NEW.name,
            'age', NEW.age,
            'favorite_toy', NEW.favorite_toy,
            'favorite_treat', NEW.favoriteTreat,
        )::text);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

I’ll explain this code briefly, but for a more in-depth page on defining functions specific to Postgres, I recommend this: https://www.postgresql.org/docs/current/sql-createfunction.html

So, here we’re creating a function called `new_dog_alert()` which will `PERFORM` a `pg_notify()` call whenever it is triggered. 

The key lines for our business logic are:

- `IF NEW.type LIKE ‘dog’ THEN` where we check that the new insert’s type is “dog”
- `PERFORM pg_notify(‘dog_lovers’, json_build_object(…)` where we run `pg_notify()` to send the message out to our channel named “dog_lovers” and build a json object which will have all of the information we supply it.
- `'operation', TG_OP` which supplies the type of operation in our JSON.


So now we have a trigger function that sends a json into the “dog_lovers” channel everytime an INSERT happens. We can make sure of this by checking the JSON and seeing that every single message has an insert. Postgresql can handle messages up to 8kb which allows for rich JSONs.

Before moving onto writing code that listens for triggers, we can listen to our messages through the psql console like this:

`LISTEN dog_lovers`

To test out this command, I edited my database of exercises in my workout tracking app so that a new message is sent out everytime a new tricep exercise is added. I’ve aptly called my channel “tricep_lovers”.

![DBeaver Console](https://antorwrites.wordpress.com/wp-content/uploads/2024/07/screenshot-2024-07-26-at-8.37.33e280afpm.png?w=1024)
*My code in DBeaver*

![Console command to Listen to my channel](https://antorwrites.wordpress.com/wp-content/uploads/2024/07/screenshot-2024-07-26-at-8.40.24e280afpm.png?w=1024)
*Console window where I listen to my channel using the psql cli*

![Console listening to channel](https://antorwrites.wordpress.com/wp-content/uploads/2024/07/screenshot-2024-07-26-at-8.48.36e280afpm.png?w=1024)

After some struggle (I forgot a semicolon after my initial “LISTEN”), you can see a message come in after I added the “Skull Crushers” exercise.
<br>More on this here: https://www.postgresql.org/docs/current/sql-listen.html

### Setting up an Application to Listen in Python
I’ve seen a number of people create a simple application like this in GO but I’m not super familiar with GO so I decided to go with Python for the sake of this tutorial. If you are curious though, here is a good post that also goes into chunking payloads: https://ds0nt.com/postgres-streaming-listen-notify-go

For this, you’ll want to have a Python virtual machine. I’ve used Python 3.12.4 here but there shouldn’t be any significant differences at least as far as Python 3.8.

Next, you’ll want to get psycopg2 (pip install psycopg2).

Now you’ll need some code to connect to your database.

```python
def create_listen_connection(self, channel_name: str = "") -> connection:
    db = psycopg2.connect(self.conn_string) # Connect to a new DB instance
     
    # I chose this isolation level for this tutorial because it allows me to be a bit more concise
    # but if require more control, I would recommend you try something else.
    # For more detail: https://www.psycopg.org/docs/extensions.html#isolation-level-constants
    db.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)
 
    cursor = db.cursor()
    if channel_name:
        cursor.execute(f"LISTEN {channel_name}")
 
    # Return the db context so you can use it in other functions
    return db
```

I’ve left some comments on important bits. This function will simply create a db connection and execute a command that listens to the channel we supply it.

Next, we simply need a function that subscribes and does something with the notifications.

```python
async def do_something_on_notify(self, db: connection) -> None:
    try:
        while True:
            # Use a select to keep watch over file descriptors
            # [db] contains fds to check if we can read db
            # empty lists in between are for writeability and exceptions
            # 5 represents a timeout
            # If we have a message available for us to read, then this condition fails
            # at that point, move into the else loop and process the message
            if select.select([db], [], [], 5) == ([], [], []):
                print("Waiting for messages...")
            else:
                db.poll()
                while db.notifies:
                    note = db.notifies.pop(0)
                    try:
                        print(f"Received message in subscribed channel: {note}")
                    except Exception as exc:
                        print(f"Unhandled exception for message in channel {self.channel}: {exc}")
            await asyncio.sleep(0.1)  # Small sleep to allow other tasks to run, otherwise this function would block others 
    except KeyboardInterrupt:
        print("Listener stopped by user.")
    finally:
        db.close()
```

So if you were to start running this in your console, you would see the following console line print out once you add Skull Crushers to your exercise DB:
> ```
> Received message in subscribed channel: "{"operation" : "INSERT", "name" : "Skull Crushers", "description" : "An exercise where you use an EZ Bar or dumbbells while laying down. You lower the weight towards your forehead in a controlled motion focusing on your triceps."}"
> ```

### Pitfalls

To be honest, I consider this a niche feature because the situations to use this would not occur very often. First of all, this is simply a built-in barebones message broker. If you need something robust, you would simply use a message broker like Kafka.

I believe the LISTEN/NOTIFY features are excellent when you are subscribing to schemas where changes don’t happen often, you have extra DB resources to spare, and you’re able to control when you poll by anticipating when changes may happen.

But… if you can anticipate when changes happen, would you need this? I can only think of a situation where a user is making a change to a schema that may be tied to another. In that case, you may want to start polling for specific changes to see if you have to invalidate your cached values somewhere. Maybe I’m thinking about this the wrong way, I’d love to hear any comments and thoughts on this.

These are some situations that I believe Postgresql LISTEN/NOTIFY will fall short:

1. **High-Frequency Events:**
   - If the events you’re tracking occur frequently, LISTEN/NOTIFY will not be efficient due to the potential overhead of sending and processing many notifications. At the end of the day, any application that subscribes to the channel is constantly polling the channel.
2. **Complex Event Processing:**
   - The features and API around polling are not robust enough for you to safely exect complex messages. Although all message brokers come with a degree of packet loss, using a dedicated event processing system like Apache Kafka, RabbitMQ, or AWS Kinesis will be more appropriate.
     - On this note, the max size of your payload via LISTEN/NOTIFY is 8KB.
3. **Cross-Database or Cross-Instance Communication:**
   - LISTEN/NOTIFY works within a single PostgreSQL database instance. For cross-database or cross-instance communication, you will need an external messaging system.
4. **Long-Running or Blocking Operations:**
   - Notifications in PostgreSQL are not meant for long-running or blocking operations.
   
PostgreSQL is a great relational database solution and I’ve enjoyed opportunities to work with the unique tools it offers. At the end of the day, I believe the LISTEN/NOTIFY is a neat feature to allow some quick integration and outputs, but if it’s something that you start to rely on then you probably need to integrate a message broker.