## Mongo-Connector Tutorial

### Mongo-Connector

I don't know anything about elasticsearch. All I know is this
[mongo-connector](https://github.com/mongodb-labs/mongo-connector) thing makes life easy. Here are some quick start
instructions to follow for setup:

### AWS ES

I've set up an Elasticsearch instance using the AWS ES service on the
free tier. I chose AWS because it didn't have arbitrary limits on
indices and what not.

### mLab (DBaaS)

For mongo, I'm using an [mLab](https://www.mlab.com/) instance, which is connected to a Heroku
app. You will need to create an oplog user to read the mongo oplog. Instructions can be found
in the [mLab docs](http://docs.mlab.com/oplog/#for-deployments-running-mongodb-v26-or-higher).
You will add its credentials to your `config.json`.

### AWS EC2

In addition to the two services, I am using a t2.micro instance to run
mongo-connector. In order to allow the t2.micro to communicate with the
ES instance, I altered the AWS access policy. It looks like this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "es:*",
      "Resource":
"arn:aws:es:us-east-1:ACOUNT_ID:domain/name-of-es-instance/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": [
            "internal.ip.of.t2micro"
          ]
        }
      }
    }
  ]
}
```

I wanted to use IAM roles, but it's not really clear to me how they'd be
used. I attached a role to my IAM user, but I still couldn't access the
instance, so I gave up on that thought.

----

I'm going to omit the step where I inserted data into the DB. If you
don't have data, then this is a sort of pointless exercise anyhow.

----

## Installation

Next, you need to install the mongo-connector software. Instructions
can be found in the mongo-connector
[README.md](https://github.com/mongodb-labs/mongo-connector#installation),
but they're pretty short, so I'll include them here too.

```command
# Assuming you use an ubuntu instance, `sudo apt-get install python-pip`
# will install pip for you
pip install mongo-connector
```

### Doc Managers

Mongo-connector has two elasticsearch doc managers available, but AWS ES
is only available in version 1.x, so you'll need to install the
respective
[elastic-doc-manager](https://github.com/mongodb-labs/elastic-doc-manager).
It's also very straightforward, so I'll include its instructions as
well.

```command
pip install elastic-doc-manager
```

### Running as a service

To start the mongo-connector, there's some bash command you can run, but
I don't know any of the syntax. Instead, I use the sweet daemonized
installation steps.

```bash
# You can install git with `sudo apt-get install git`
git clone git@github.com:mongodb-labs/mongo-connector.git

# Edit the config.json to your liking
vim config.json
```

What is 'your liking'? I've included an example similar to the one I
use, but you can learn what all the fields mean by reading the docs on
the [Configuration
Options](https://github.com/mongodb-labs/mongo-connector/wiki/Configuration-Options) page of their wiki.

Once your config is good, do the following:

```command
# Installs as daemonized service
python setup.py install_service

# Starts service
service mongo-connector start
```

If you take a look in the mongo-connector logs, you should see a
bunch of information getting spat out. This is a sign that it's syncing!
To verify that the mongo collections have been sync'd to the
elasticsearch instance, you can take a look at the indices tab for your
AWS ES instance. It should have an index that is the same name as your
mongo db and a caret that opens up and shows a list of mappings (your
mongo collections).

## Performing some Queries

If you'd like to run some test queries, I recommend using
[POSTMan](https://chrome.google.com/webstore/detail/postman-rest-client-short/mkhojklkhkdaghjjfdnphfphiaiohkef).
Worth noting, elasticsearch is a little weird because most advanced
queries are performed with a POST instead of a GET. Maybe it's more
subjective, than it is weird, but whatever. Here's what a query would
look like for a car document that looks like this:

```json
{
    "brand": "Tesla",
    "model": "Model 3",
    "perks": "safe, aerodynamic, electric"
}
```

`name-of-aws-es-instance-and-some-gobbledy-gook.us-east-1.es.amazonaws.com/database_name/cars/_search/?size=15`

and the payload would be:

```json
{
  "query": {
    "multi_match": {
      "fields":  [ "brand", "perks"],
      "query":     "aerodinamic",
      "fuzziness": "AUTO"
    }
  }
}
```

Once you POST to the URL with the above payload, elasticsearch will
perform a fuzzy search to get your results. It's pretty cool.

That's all! :smile:
