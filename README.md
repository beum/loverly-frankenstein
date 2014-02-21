# loverly-frankenstein


The Loverly node.js model layer that allows a unified hierarchical structure on
top of an assortment of data-storage engines.

Tired of that restrictive flat relational structure?  Want a nice JSON-based
document on top of a legacy MySQL data store? No problem! __loverly-frankenstein__
is here to save the day!

The goal of this module is to abstract the underlying data storage engines from
the end-user's interactions, creating a clean easy-to-use presentation layer that
can fit into a standardized REST API.


## Is there a difference between an ORM and loverly-frankenstein?

Conceptually, not really.  At its core, the goal is the same, take data from a data
storage system and transform it into an object that is relevant to the application.
loverly-frankenstein allows you to specify in a very fine-grained fashion how
various pieces of data should be sewn together to form a cohesive system entity.


# Contents

* [Usage](#usage)
* [Model concepts](#model-concepts)
* [Creating a model](#creating-a-model)
* [Creating your own data sources](#creating-your-own-data-sources)


# Usage

Install in typical fashion:

    npm install loverly-frankenstein --save-dev


## Setting up loverly-frankenstein

The configuration and management of models can become quite complex if managed
manually.  I prefer to use a [Service Container](https://github.com/linkshare/service-container)
to manage building the objects I need, but you can do it manually if you prefer.


__TODO__: Some nice example here



# Model concepts

From an application perspective, all interaction with your various data sources
should occur through Instances of a loverly-frankenstein __model__.  If you've
used an ORM like [Sequelize.js](http://sequelizejs.com/) or an ODM like [Mongoose](http://mongoosejs.com/)
then you're probably familiar with the concept.  A model represents an entity in
your system.  This could be a User, a Product, a Car, or anything that is relevant
to you and your application.  Sometimes the data for a specific entity is spread
across multiple data sources and is fairly hard to deal with consistently and easily.

Take for example, a User.  A User may have typical things like a username and a password,
but in a more complicated/broadly integrated system, they probably have Facebook
friends, Twitter followers, Instagram photos, analytics information, etc.  Wouldn't
it be nice to have all of these disparate pieces of information be gathered together
in a single object:

```javascript
// My user object
{
  name: "brandon",
  username: "brandon@lover.ly",
  password: "f234oiiO0334Fwer",
  facebook: {
    friends: [{id: 1, name: "alice"}, {id: 2, name: "bob"}]
  },
  twitter: {
    followers: [{id: 1, name: "alice"}, {id: 2, name: "bob"}]
  },
  instagram: {
    photos: [{id: 27, src: "http://coolimage.com"}]
  }
}
```

Upon inspection, each piece of data comes from a different system/API.  The basic
account info may come from our own system (a DB, perhaps mysql, mongo, etc), the
facebook, twitter, and instagram info probably comes from their public APIs.

The goal of loverly-frankenstein is to give you an easy-to-use interface to deal
with multiple data sources as a single object. Imagine a world where you could
manipulate all of the various data sources as part of the same object and then
simply call:

    user.save(callback);

And just like that, all of the various API's and DB's where called in the right
order and with the right calls.  Too good to be true?  Probably.  That's why this
library's not done yet :P


## What is a model?

Like the above example, a model is a representation of your data.  It contains a
specific structure, and provides a consistent API for reading and listing instances
of that particular model.

The magic of a model is that fields can be re-organized and aliased into new names
to suit your particular application needs.  For example this DB record:

    {name: "brandon", joined: "2014-01-01"}

Can become:

    {full_name: "brandon", important_dates: {joined: {year: 2014, month: 1, day: 1}}}

with a few simple manipulations.  This is especially useful in legacy applications
where data has non-sensical names or structures that must be tolerated for some
transition period before it can be reorganized into a better schema.

Possibly more useful, it also provides the ability to abstract an arbitrary
number of disparate data sources and manage the retrieval and manipulation of
data. A call like:

    User.read(query, options, callback);

Should be able to make the necessary number of calls to its data sources (whether
one or many) to ensure that the requested information is available.

Upon saving or updating, the model's instance should also be capable of assigning
the correct fields of information to each data source and requesting that they
create/update/delete as necessary.  The interface should be a simple CRUD API and
should completely decouple the storage and management of data from its use within
the application.

I guarantee that in both the short and long term it will make your data easier
to manage and easier to move/change when necessary.


## Everything is a Data Source

Models in loverly-frankenstein are retrieve and manipulate their data via their
data sources.  Each data source is responsible for a set number of fields within
a model and each source is treated as separate and unrelated to other data sources
except via the model.

A data source can be:

* A table in a DB
* An web API
* A file on disk
* An object in memory
* Another model!

An interesting design element is that models implement the same API as data sources
do, and can therefore be used as data sources for other higher-level models.  This
isn't a great idea, as the number of operations needed to retrieve data for a
model will probably vastly increase as you "frankenstein" more of these models
together, however it is useful for creating distinct views and combinations of
existing data.


## The price of abstraction

Efficiency is always the first sacrifice in a generally-applicable tool.  Many
performance tweaks rely on knowledge of a particular API or DB system in order
to optimize a specific operation.  To give a specific example, every data source is
treated as logically separate (except through defined key-based relationships)
and have no interaction together.  Therefore when we define relationships between
multiple tables in a DB it takes more work to create the same join statement that
direct querying would easily enable.

These kind of limitations can be circumvented by defining a higher-level Data Source
that is represented by the joining of the multiple tables, but this comes with
cost of increased complexity and slightly more difficult maintenance (for the
sake of performance).

When dealing with a legacy system or a system you do not control, you may be forced
to marry many disparate sources of information, resulting in some less than optimal
querying.  Over time, I would hope that your system would evolve closer and closer
to the "ideal" models that your application uses and therefore result in as few
and as efficient requests as possible to manipulate your data.


# Creating a model

When creating a new model, I like to begin with the "ideal" design of your data.
What does this model represent?  How will it be used in your application?  How
does it relate to other models?

A model should be defined in your own domain specific language (DSL) and should
use terms that mean something to you and members of your team.  Be very careful
when you name a model or its fields, you could be stuck with them for a long time.

A very simple model might look like this:

```javascript
var AbstractModel = require('loverly-frankenstein').Model;

/**
 * A model for a car
 *
 * @class CarModel
 * @constructor
 * @extends {AbstractModel}
 */
var CarModel = function () {
  this.name = 'Car';

  this.definition = {};

  /**
   * The name of the manufacturer
   *
   * @property definition.manufacturer
   * @type {String}
   */
  this.definition.manufacturer = {
    "type": this.TYPES.STRING,
    "views": ["default"],
    "required": true,
    "constraints": {
      "isLength": {
        "msg": "The manufacturer's name was too long",
        "args": [0, 50]
      }
    }
  };

  /**
   * The model number of the car
   *
   * @property definition.model_number
   * @type {Int}
   */
  this.definition.model_number = {
    "type": this.TYPES.INTEGER,
    "views": ["default"],
    "required": true
  };

  this.sources = {};

  /**
   * A table in a relational DB that stores cars.
   *
   * @property sources.CarsTable
   * @type {AbstractTable}
   */
  this.sources.CarsTable = {
    "relationship": this.SOURCE_TYPES.ONE_TO_ONE,
    "is_primary": true
  };

  AbstractModel.call(this);
};

CarModel.prototype = new AbstractModel();
```

A couple things to note about the example above: I use YUI docs for creating
my API docs which I find extremely helpful for maintaining a long-term understanding
of what each field in a model does.  I am using one particular style of prototypal
inheritance but it is also possible to achieve the same thing via:

(`Model.create()` NOT YET IMPLEMENTED)

```javascript
// CarModel.js

var frankenstein = require('loverly-frankenstein');
var TYPES = frankenstein.TYPES;
var Model = frankenstein.Model;

var definition = {
  // Same as above...
};

var sources = {
  // Same as above...
};

module.exports = Model.create('Car', definition, sources);
```

I prefer the more manual approach because it allows for better documentation and
control but the choice is yours.

I will go over the example step by step in the sections below.


## Definition

The definition object defines the structure of the data that your app will receive
and that your API (if you choose to implement one) will serve. Each property within
the definition represents a field within the output.

## Data types

* NUMBER
* INTEGER
* STRING
* DATE
* BOOLEAN
* ARRAY


## Field mapping types

## Views


## Constraints


## Nested objects


## Nested models



## Data Sources
### One-to-one relationships

### One-to-many relationships

### Many-to-many relationships


# Creating your own data sources
## Integrating an external API


## Setting up the CRUD routes

# Examples

# Run the Tests
