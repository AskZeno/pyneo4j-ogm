# Neo4j Object Graph Mapper for Python

A asynchronous library for working with Neo4j and Python 3.10+. This library aims to solve the problem of repeating the same queries over and over again by providing a simple way to define your models and their relationships. It is built on top of the [`Neo4j Python Driver`](https://neo4j.com/docs/api/python-driver/current/index.html) and [`Pydantic 1.10`](https://docs.pydantic.dev/1.10/).

## Todo's

- [ ] Add more documentation
- [ ] Add more tests
- [ ] Add setting for automatic create/update timestamps for models, timestamps will be overwritten if also defined on model
- [ ] Make warnings optional (for example when using the name of the model as the label/type) via model setting
- [ ] Add migrations
- [ ] Make internal ID available and raise Exception if a ID property gets defined
- [ ] Set a requirement for min. Neo4j database version
- [ ] Add upsert-like functionality to find_one()
- [ ] Add MIT license
- [ ] Add some metadata to pyproject.toml before first release
- [ ] Add support for Pydantic v2

## ⚡️ Quick start <a name="quick-start"></a>

Before we can start to interact with our database, we have to take care of some setup. First, we have to connect to our database with the `connect()` method from the `Neo4jClient` class. This can be done by creating a new instance of the `Neo4jClient` class and passing the connection details as keyword arguments. This class is our entry point to the library and will be used to interact with our database.

```python
from pyneo4j_ogm import Neo4jClient

# Initialize the client class and call the `.connect()` method
client = Neo4jClient()
client.connect(uri="<connection-uri-to-database>", auth=("<username>", "<password>"))

# The `.connect()` allows to be chained right after the instantiation of the class
# for more convenience.
client = Neo4jClient().connect(uri="<connection-uri-to-database>", auth=("<username>", "<password>"))
```

Next, we need to define our models. Models are used to represent data in our database. They are defined by inheriting from either the `NodeModel` or `RelationshipModel` class, depending on the type of graph entity we want our data to represent. Let's create a `Developer` who `DRINKS` some `Coffee`:

```python
from pyneo4j_ogm import (
  NodeModel,
  RelationshipModel,
  RelationshipProperty,
  RelationshipPropertyDirection,
  RelationshipPropertyCardinality
)


class Developer(NodeModel):
  """
  A model representing a developer node in the graph.
  """
  name: str
  age: int

  coffee: RelationshipProperty["Coffee", "Drinks"] = RelationshipProperty(
    target_model="Coffee",
    relationship_model="Drinks",
    direction=RelationshipPropertyDirection.OUTGOING,
    cardinality=RelationshipPropertyCardinality.ZERO_OR_MORE,
    allow_multiple=True,
  )


class Coffee(NodeModel):
  """
  A model representing a coffee node in the graph.
  """
  name: str


class Drinks(RelationshipModel):
  """
  A model representing a DRINKS relationship between a developer and a coffee node.
  """
  likes_it: bool
```

Now that we have our models defined, we have to let our client know about them. This can be done by calling the `register_models()` method of the `Neo4jClient` class and passing our models as a list.

```python
# Client initialization and model definition happened here ...

# Register the models with the client
await client.register_models([Developer, Coffee, Drinks])
```

> **Note**: This step can be skipped in certain cases, but it is recommended to always register your models with the client. Skipping this step will result in you loosing a big portion of the library's functionality (Like working with your models).

Finally, we can start to work with our database. Let's create a new `Developer` node and a `Coffee` node and connect them with a `DRINKS` relationship.

```python
# Create a new developer node
developer = await Developer(name="John Doe", age=25).create()

# Create a new coffee node
coffee = await Coffee(name="Espresso").create()

# Connect the developer and coffee nodes with a DRINKS relationship
await developer.coffee.connect(coffee, {"likes_it": True})
```

So if we put it all together, our code should look something like this:

```python
import asyncio

from pyneo4j_ogm import (
    Neo4jClient,
    NodeModel,
    RelationshipModel,
    RelationshipProperty,
    RelationshipPropertyCardinality,
    RelationshipPropertyDirection,
)


class Developer(NodeModel):
    """
    A model representing a developer node in the graph.
    """

    name: str
    age: int

    coffee: RelationshipProperty["Coffee", "Drinks"] = RelationshipProperty(
        target_model="Coffee",
        relationship_model="Drinks",
        direction=RelationshipPropertyDirection.OUTGOING,
        cardinality=RelationshipPropertyCardinality.ZERO_OR_MORE,
        allow_multiple=True,
    )


class Coffee(NodeModel):
    """
    A model representing a coffee node in the graph.
    """

    name: str


class Drinks(RelationshipModel):
    """
    A model representing a DRINKS relationship between a developer and a coffee node.
    """

    likes_it: bool


async def main():
    # Connect to the database and register the models
    client = Neo4jClient().connect(uri="bolt://localhost:7687", auth=("neo4j", "password"))
    await client.register_models([Developer, Coffee, Drinks])

    # Create new developer and coffee nodes and connect them with a DRINKS relationship
    developer = await Developer(name="John Doe", age=25).create()
    coffee = await Coffee(name="Espresso").create()

    await developer.coffee.connect(coffee, {"likes_it": True})

    # Close connection again
    await client.close()


# Run the main function
asyncio.run(main())
```

> **Note**: This script should run `as is`. You must change the `uri` and `auth` parameters in the `connect()` method call to match the ones needed for your own database before starting the script.

## 📚 Documentation <a name="documentation"></a>

### Table of contents <a name="table-of-contents"></a>

- [Neo4j Object Graph Mapper for Python](#neo4j-object-graph-mapper-for-python)
  - [Todo's](#todos)
  - [⚡️ Quick start ](#️-quick-start-)
  - [📚 Documentation ](#-documentation-)
    - [Table of contents ](#table-of-contents-)
    - [Basic concepts ](#basic-concepts-)
    - [Database client ](#database-client-)
      - [Connecting to a database ](#connecting-to-a-database-)
      - [Closing an existing connection ](#closing-an-existing-connection-)
      - [Registering models ](#registering-models-)
      - [Executing Cypher queries ](#executing-cypher-queries-)
      - [Batching cypher queries ](#batching-cypher-queries-)
      - [Manual indexing and constraints ](#manual-indexing-and-constraints-)
      - [Client utilities ](#client-utilities-)
    - [Model configuration ](#model-configuration-)
      - [Defining node and relationship properties ](#defining-node-and-relationship-properties-)
      - [Defining indexes and constraints on model properties ](#defining-indexes-and-constraints-on-model-properties-)
      - [Defining settings for a model ](#defining-settings-for-a-model-)
        - [NodeModel specific settings ](#nodemodel-specific-settings-)
        - [RelationshipModel specific settings ](#relationshipmodel-specific-settings-)
        - [How to use model settings ](#how-to-use-model-settings-)
    - [Model methods ](#model-methods-)
      - [Instance.update() ](#instanceupdate-)
      - [Instance.delete() ](#instancedelete-)
      - [Instance.refresh() ](#instancerefresh-)
      - [Model.find\_one() ](#modelfind_one-)
        - [Projections ](#projections-)
        - [Auto-fetching nodes ](#auto-fetching-nodes-)
      - [Model.find\_many() ](#modelfind_many-)
        - [Filters ](#filters-)
        - [Projections ](#projections--1)
        - [Auto-fetching nodes ](#auto-fetching-nodes--1)
      - [Model.update\_one() ](#modelupdate_one-)
        - [Returning the updated entity ](#returning-the-updated-entity-)
      - [Model.update\_many() ](#modelupdate_many-)
        - [Filters ](#filters--1)
        - [Returning the updated entity ](#returning-the-updated-entity--1)
      - [Model.delete\_one() ](#modeldelete_one-)
      - [Model.delete\_many() ](#modeldelete_many-)
          - [Filters ](#filters--2)
      - [Model.count() ](#modelcount-)
        - [Filters ](#filters--3)
      - [Instance.export\_model() ](#instanceexport_model-)
        - [Case conversion ](#case-conversion-)
      - [Instance.import\_model() ](#instanceimport_model-)
        - [Case conversion ](#case-conversion--1)
      - [Model.register\_pre\_hooks() ](#modelregister_pre_hooks-)
        - [Overwriting existing hooks ](#overwriting-existing-hooks-)
      - [Model.register\_post\_hooks() ](#modelregister_post_hooks-)
      - [Model.model\_settings() ](#modelmodel_settings-)

### Basic concepts <a name="basic-concepts"></a>

If you have worked with other ORM's like `sqlalchemy` or `beanie` before, you will find that this library is very similar to them. The main idea is to work with nodes and relationships in a **structured and easy-to-use** way instead of writing out complex cypher queries and tons of boilerplate for simple operations.

The concept of this library builds on the idea of defining nodes and relationships present in the graph database as **Python classes**. This allows for easy and structured access to the data. These classes provide a robust foundation for working with a Neo4j database using methods for common queries, a filter API, model resolution and more.

In the following sections we will take a look at all of the features this library has to offer and how to use them.

> **Note:** All of the examples in this documentation assume that you have already connected to a database and registered your models with the client instance. The models used in the following examples will build upon the ones defined in the [`Quick start`](#quick-start) section.

### Database client <a name="database-client"></a>

The `Neo4jClient` class is the brains of the library. It is used to connect to a database instance, register models and execute queries. Under the hood, every class uses the client to execute queries.

#### Connecting to a database <a name="connecting-to-a-database"></a>

Before you can interact with anything this library offers in any way, you have to connect to a database. You can do this by creating a new instance of the `Neo4jClient` class and calling the `connect()` method on it. The `connect()` method takes a few arguments:

- `uri`: The connection URI to the database.
- `skip_constraints`: Whether the client should skip creating any constraints defined on models when registering them. Defaults to `False`.
- `skip_indexes`: Whether the client should skip creating any indexes defined on models when registering them. Defaults to `False`.
- `*args`: Additional arguments that are passed directly to Neo4j's `AsyncDriver.driver()` method.
- `**kwargs`: Additional keyword arguments that are passed directly to Neo4j's `AsyncDriver.driver()` method.

The `connect()` method returns the client instance itself, so you can chain it right after the instantiation of the class. Here is an example of how to connect to a database:

```python
from pyneo4j_ogm import Neo4jClient

client = Neo4jClient()
client.connect(uri="<connection-uri-to-database>", auth=("<username>", "<password>"), max_connection_pool_size=10, ...)

# Or chained right after the instantiation of the class
client = Neo4jClient().connect(uri="<connection-uri-to-database>", auth=("<username>", "<password>"), max_connection_pool_size=10, ...)
```

After connecting the client, you will be able to run queries against the database. Should you try to run a query without connecting to a database first, you will get a `NotConnectedToDatabase` exception.

#### Closing an existing connection <a name="closing-an-existing-connection"></a>

Once you are done working with a database and the client is no longer needed, you can close the connection to it by calling the `close()` method on the client instance. This will close the connection to the database and free up any resources used by the client. Here is an example of how to close a connection to a database instance:

```python
# Do some work with the database instance ...

await client.close()
```

Once you closed the client, it will be seen as `disconnected` and if you try to run any further queries with it, you will get a `NotConnectedToDatabase` exception.

#### Registering models <a name="registering-models"></a>

As mentioned before, the basic concept is to work with models which reflect the nodes and relationships inside the graph. In order to work with these models, you have to register them with the client. You can do this by calling the `register_models()` method on the client and passing in your models as a list. Let's take a look at an example:

```python
# Create a new client instance and connect ...

await client.register_models([Developer, Coffee, Drinks])
```

This is a crucial step, because if you don't register your models with the client, you won't be able to work with them in any way. Should you try to work with a model that is not registered with the client instance, you will get a `UnregisteredModel` exception. This exception also gets raised if a database model defines a relationship property with other (unregistered) models as a target or relationship model. For more information about relationship properties, see the [`Relationship properties on node models`](#relationship-properties-on-node-models) section.

> **Note**: If you don't explicitly define labels for node models or a type for a relationship model, you will see a warning in the console. This is because the library will use the name of the model as the label/type. This is not recommended, because most of the time you will want to define your own labels/types.

If you have defined any indexes or constraints on your models, they will be created automatically when registering them (if not disabled when the client got initialized).

If you don't register your models with the client, you will still be able to run cypher queries directly with the client, but you will `loose automatic model resolution` from queries. This means that, instead of database models, the raw Neo4j classes are returned.

#### Executing Cypher queries <a name="executing-cypher-queries"></a>

Node- and RelationshipModels provide many methods for commonly used cypher queries, but sometimes you might want to execute a custom cypher with more complex logic. For this purpose, the client instance provides a `cypher()` method that allows you to execute custom cypher queries. The `cypher()` method takes three arguments:

- `query`: The cypher query to execute.
- `parameters`: A dictionary containing the parameters to pass to the query.
- `resolve_models`: Whether the client should try to resolve the models from the query results. Defaults to `True`.

This method will always return a tuple containing list of result lists and a list of variables returned by the query. Internally, the client uses the `.values()` method of the Neo4j driver to get the results of the query.

> **Note:** If no models have been registered with the client and resolve_models is set to True, the client will not raise any exceptions but rather return the raw query results.

Here is an example of how to execute a custom cypher query:

```python
# Create a new client instance and connect ...

results, meta = await client.cypher(
  query="MATCH (d:Developer) WHERE d.name = $name RETURN d.name as developer_name, d.age",
  parameters={"name": "John Doe"},
  resolve_models=False, # Explicitly disable model resolution
)

print(results) # [["John Doe", 25]]
print(meta) # ["developer_name", "d.age"]
```

#### Batching cypher queries <a name="batching-cypher-queries"></a>

Since Neo4j is ACID compliant, it is possible to batch multiple cypher queries into a single transaction. This can be useful if you want to execute multiple queries at once and make sure that either all of them succeed or none of them do. The client provides a `batch()` method that allows you to batch multiple cypher queries into a single transaction. The `batch()` method has to be called with a asynchronous context manager like in the following example:

```python
# Create a new client instance and connect ...
# Make sure to register models when using them!

async with client.batch():
  # All queries executed inside the context manager will be batched into a single transaction
  # and executed once the context manager exits. If any of the queries fail, the whole transaction
  # will be rolled back.
  await client.cypher(
      query="CREATE (d:Developer {name: $name, age: $age})",
      parameters={"name": "John Doe", "age": 25},
  )
  await client.cypher(
      query="CREATE (c:Coffee {name: $name})",
      parameters={"name": "Espresso"},
  )

  # Model queries also can be batched together
  coffee = await Coffee(name="Americano").create()
```

#### Manual indexing and constraints <a name="manual-indexing-and-constraints"></a>

By default, the client will automatically create any indexes and constraints defined on models when registering them. If you want to disable this behavior, you can do so by passing the `skip_indexes` and `skip_constraints` arguments to the `connect()` method when connecting your client to a database.

If you want to create custom indexes and constraints, or want to add additional indexes/constraints later on (which should probably be done on the models themselves), you can do so by calling the `create_index()` and `create_constraint()` methods on the client.

First, let's take a look at how to create a custom index in the database. The `create_index()` method takes a few arguments:

- `name`: The name of the index to create (Make sure this is unique!).
- `entity_type`: The entity type the index is created for. Can be either **EntityType.NODE** or **EntityType.RELATIONSHIP**.
- `index_type`: The type of index to create. Can be `IndexType.RANGE`, `IndexType.TEXT`, `IndexType.POINT` or `IndexType.LOOKUP`.
- `properties`: A list of properties to create the index for.
- `labels_or_type`: The node labels or relationship type the index is created for.

The `create_constraint()` method also takes similar arguments as the `create_index()` method. The only difference is that it does not take a `constraint_type` argument since only `UNIQUENESS` constraints are currently supported.

- `name`: The name of the constraint to create.
- `entity_type`: The entity type the constraint is created for. Can be either **EntityType.NODE** or **EntityType.RELATIONSHIP**.
- `properties`: A list of properties to create the constraint for.
- `labels_or_type`: The node labels or relationship type the constraint is created for.

Here is an example of how to use the methods:

```python
# Creates a `RANGE` index on the labels `Test` and `Node`
# for both properties `prop_a and prop_b`
await client.create_index("node_text_index", EntityType.NODE, IndexType.RANGE, ["prop_a", "prop_b"], ["Test", "Node"])

# Creates a UNIQUENESS constraint for properties `prop_a and prop_b`
# for all relationships of type `TEST_RELATIONSHIP`
await client.create_constraint("relationship_constraint", EntityType.RELATIONSHIP, ["prop_a", "prop_b"], "TEST_RELATIONSHIP")
```

#### Client utilities <a name="client-utilities"></a>

The database client also provides a few utility methods and properties that can be useful when writing automated scripts or tests. These methods are:

- `is_connected()`: Returns whether the client is currently connected to a database.
- `drop_nodes()`: Drops all nodes from the database.
- `drop_constraints()`: Drops all constraints from the database.
- `drop_indexes()`: Drops all indexes from the database.

### Model configuration <a name="model-configuration"></a>

Both Node- and RelationshipModels provide a few configuration options that can be used to define indexes, constraints and other settings for the model. These options can be used to easily define how nodes and relationships are created in the database.

Since this library uses `pydantic` under the hood, all of the configuration options available for pydantic models are also available for Node- and RelationshipModels. For more information about the configuration options available for pydantic models, see the [`pydantic docs`](https://docs.pydantic.dev/1.10/).

#### Defining node and relationship properties <a name="defining-node-and-relationship-properties"></a>

Node- and RelationshipModels are defined in the same way as pydantic models. The only difference is that you have to use the `NodeModel` and `RelationshipModel` classes instead of pydantic's `BaseModel` class. Here is an example of how to define a simple NodeModel:

```python
from pyneo4j_ogm import NodeModel
from pydantic import Field, validator
from uuid import UUID, uuid4

class Developer(NodeModel):
  """
  A model representing a developer node in the graph.
  """
  id: UUID = Field(default_factory=uuid4)
  name: str
  age: int
  likes_his_job: bool

  @validator("likes_his_job")
  def must_like_his_job(cls, v):
    if not v:
      raise ValueError("A developer must like his job!")
    return v
```

A node created with the above model could look like this as a cypher query:

```cypher
CREATE (d:Developer {id: "a1b2c3d4-e5f6-g7h8-i9j0-k1l2m3n4o5p6", name: "John Doe", age: 25, likes_his_job: true})
```

> **Note:** Everything mentioned above also applies to `RelationshipModels`.

#### Defining indexes and constraints on model properties <a name="defining-indexes-and-constraints-on-model-properties"></a>

This library provides a convenient way to define indexes and constraints on model properties. This can be done by using the `WithOptions` method instead of the datatype when defining the properties of your model. The `WithOptions` method takes a few arguments:

- `property_type`: The datatype of the property.
- `range_index`: Whether to create a range index on the property. Defaults to `False`.
- `text_index`: Whether to create a text index on the property. Defaults to `False`.
- `point_index`: Whether to create a point index on the property. Defaults to `False`.
- `unique`: Whether to create a uniqueness constraint on the property. Defaults to `False`.

If the method is called without defining any indexes or constraints, it behaves just like any other property. Here is an example of how to define indexes and constraints on model properties:

```python
from pyneo4j_ogm import NodeModel, WithOptions
from pydantic import Field
from uuid import UUID, uuid4

class Developer(NodeModel):
  """
  A model representing a developer node in the graph.
  """
  id: WithOptions(UUID, unique=True) = Field(default_factory=uuid4)
  name: WithOptions(str, text_index=True)
  age: int
  likes_his_job: WithOptions(bool) # Has no effect since no indexes or constraints are defined
```

In the example above, we defined a `text index` on the `name` property and a `uniqueness constraint` on the `id` property. If you want to define multiple indexes or constraints on a single property, you can do so by passing multiple arguments to the `WithOptions` method. Note that you can still use the full power of pydantic like validators and default values when defining indexes and constraints on model properties.

> **Note:** Everything mentioned above also applies to `RelationshipModels`.

#### Defining settings for a model <a name="defining-settings-for-a-model"></a>

It is possible to define a number of settings for a model. These settings can be used to configure how nodes and relationships are created in the database and how you can interact with them. Settings are defined with a nested class called `Settings`. Class attributes act as the settings defined on the model. The following settings are available for Node- and RelationshipModels:

| Setting name          | Type                          | Description                                                                                                                                                                                                                                                                                                                              |
| --------------------- | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `exclude_from_export` | **Set[str]**                  | Can be used to always exclude a property from being exported when using the `export_model()` method on a model. Any `RelationshipProperty attributes` on a model are `automatically added` (only for node models). Defaults to `set()`.                                                                                                  |
| `pre_hooks`           | **Dict[str, List[Callable]]** | A dictionary where the key is the name of the method for which to register the hook and the value is a list of hook functions. The hook function can be sync or async. All hook functions receive the exact same arguments as the method they are registered for and the current model instance as the first argument. Defaults to `{}`. |
| `post_hooks`          | **Dict[str, List[Callable]]** | Same as **pre_hooks**, but the hook functions are executed after the method they are registered for. Additionally, the result of the method is passed to the hook as the second argument. Defaults to `{}`.                                                                                                                              |

##### NodeModel specific settings <a name="node-model-specific-settings"></a>

NodeModels have two additional settings, `labels` and `auto_fetch_nodes`.

| Setting name       | Type         | Description                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------ | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `labels`           | **Set[str]** | A set of labels to use for the node. If no labels are defined, the name of the model will be used as the label. Defaults to the `model name` split by `_`.                                                                                                                                                                                                                            |
| `auto_fetch_nodes` | **bool**     | Whether to automatically fetch nodes of defined relationship properties when getting a model instance from the database. Auto-fetched nodes are available at the `instance.<relationship-property>.nodes` property. If no specific models are passed to a method when this setting is set to `True`, nodes from all defined relationship properties are fetched. Defaults to `False`. |

> **Note:** When no labels are defined for a NodeModel, the name of the model will be used and converted to **PascalCase** to stay in line with the style guide for cypher. Additionally, you will receive a warning in the console (since most of the time it's not the best to use the model name as a label).

##### RelationshipModel specific settings <a name="relationshipmodel-specific-settings"></a>

For RelationshipModels, the `labels` setting is not available, since relationships don't have labels in Neo4j. Instead, the `type` setting can be used to define the type of the relationship. If no type is defined, the name of the model name will be used as the type.

| Setting name | Type    | Description                                                                                                                        |
| ------------ | ------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `type`       | **str** | The type of the relationship to use. If no type is defined, the model name will be used as the type. Defaults to the `model name`. |

> **Note:** When no type is defined for a RelationshipModel, the name of the model will be used and converted to **SCREAMING_SNAKE_CASE** to stay in line with the style guide for cypher. Additionally, you will receive a warning in the console (since most of the time it's not the best to use the model name as a type).

##### How to use model settings <a name="how-to-use-model-settings"></a>

Now, after we have taken a look at all of the available settings, let's take a look at how to use them:

```python
class Developer(NodeModel):
  ...

  coffee: RelationshipProperty["Coffee", "Drinks"] = RelationshipProperty(
    target_model="Coffee",
    relationship_model="Drinks",
    direction=RelationshipPropertyDirection.OUTGOING,
    cardinality=RelationshipPropertyCardinality.ZERO_OR_MORE,
    allow_multiple=True,
  )

  class Settings:
    # Nodes of this model will have the labels `Developer` and `Python`
    labels = {"Developer", "Python"}
    # Coffee nodes connected a Developer node will always be fetched
    auto_fetch_nodes = True


class Coffee(NodeModel):
  ...

  class Settings:
    # Since no labels have been defined, the labels for nodes of this class will
    # be `Coffee`

    # The `name` property will always be excluded from being exported
    exclude_from_export = {"name"}


def print_drink(self, result, *args, **kwargs):
  # Will print the result from the `find_one()` method
  print(result)


class Drinks(RelationshipModel):
  ...

  class Settings:
    # The type of the relationship will be `DRINKS_LOTS_OF`
    type = "DRINKS_LOTS_OF"
    # The `print_drink` method will be executed after the `find_one` method
    post_hooks = {"find_one": [print_drink]}
```

### Model methods <a name="model-methods"></a>

Node- and RelationshipModels provide a number of methods for commonly used cypher queries. In the following sections we will take a look at all of the methods available on Node- and RelationshipModels.

> **Note:** If a method is defined using **Instance.method()**, the method is to be called on a instance of a model. If a method is defined using **Model.method()**, the method is to be called on the model itself. Should a method only be available for either a NodeModel or RelationshipModel class, it will be defined as **NodeModel.method()** or **RelationshipModel.method()**.

#### Instance.update() <a name="instance-update"></a>

The `update()` method can be used to sync the modified properties of a node or relationship model with the corresponding entity inside the graph. This method takes no arguments.

```python
developer = await Developer(
  name="John",
  age=24
).create()

# Update instance with new values
developer.age = 27

# Syncs node in graph with properties from local instance
await developer.update()
```

#### Instance.delete() <a name="instance-delete"></a>

The `delete()` method can be used to delete the graph entity tied to the current model instance. This method takes no arguments and returns nothing. Once deleted, the model instance will be marked as `destroyed` and any further operations on it will raise a `InstanceDestroyed` exception.

```python
developer = await Developer(
  name="John",
  age=24
).create()

# Deletes node in database, instance is now seen as `destroyed`
await developer.delete()

developer.age = 27
await developer.update() # Raises a `InstanceDestroyed` exception
```

#### Instance.refresh() <a name="instance-refresh"></a>

Syncs your local instance with the properties from the corresponding graph entity. This method takes no arguments. This method can be useful if you want to make sure that your local instance is always up-to-date with the graph entity and should always be called when importing a model instance from a dictionary.

```python
developer = await Developer(
  name="John",
  age=24
).create()

# Something on the local instance changes
developer.age = 27
print(developer.age)  # 27

await developer.refresh()

print(developer.age)  # 24
```

#### Model.find_one() <a name="model-find-one"></a>

The `find_one()` method can be used to find a single node or relationship in the graph. If multiple results are matched, the first one is returned. This method returns a single instance/dictionary or `None` if no results were found.

This method takes a mandatory `filters` argument, which is used to filter the results. For more about filters, see the [`Filtering queries`](#query-filters) section.

```python
# Return the first encountered node where the name property equals `John`
developer = await Developer.find_one({"name": "John"})

print(developer) # <Developer>

# Or if no match was found
print(developer) # None
```

##### Projections <a name="model-find-one-projections"></a>

`Projections` can be used to only return specific parts of the model as a dictionary. This can be useful if you only need a few properties of a model and don't want to fetch the whole model from the database. For more information about projections, see the [`Projections`](#query-projections) section.

```python
# Return a dictionary with the developers name at the `dev_name` key instead
# of a model instance
developer = await Developer.find_one({"name": "John"}, {"dev_name": "name"})

print(developer) # {"dev_name": "John"}
```

##### Auto-fetching nodes <a name="model-find-one-auto-fetching-nodes"></a>

The `auto_fetch_nodes` and `auto_fetch_models` parameters can be used to automatically fetch all or selected nodes from defined relationship properties when running the `find_one()` query. For more about auto-fetching, see [`Auto-fetching relationship properties`](#query-auto-fetching).

```python
# Returns a developer instance with `instance.<property>.nodes` properties already fetched
developer = await Developer.find_one({"name": "John"}, auto_fetch_nodes=True)

print(developer.coffee.nodes) # [<Coffee>, <Coffee>, ...]
print(developer.other_property.nodes) # [<OtherModel>, <OtherModel>, ...]

# Returns a developer instance with only the `instance.coffee.nodes` property already fetched
developer = await Developer.find_one({"name": "John"}, auto_fetch_nodes=True, auto_fetch_models=[Coffee])

# Auto-fetch models can also be passed as strings
developer = await Developer.find_one({"name": "John"}, auto_fetch_nodes=True, auto_fetch_models=["Coffee"])

print(developer.coffee.nodes) # [<Coffee>, <Coffee>, ...]
print(developer.other_property.nodes) # []
```

> **Note**: The `auto_fetch_nodes` and `auto_fetch_models` parameters are only available for classes which inherit from the `NodeModel` class.

#### Model.find_many() <a name="model-find-many"></a>

The `find_many()` method can be used to find multiple nodes or relationships in the graph. This method always returns a list of instances/dictionaries or an empty list if no results were found.

##### Filters <a name="model-find-many-filters"></a>

Just like the `find_one()` method, the `find_many()` method also takes (optional) filters. For more about filters, see the [`Filtering queries`](#query-filters) section.

```python
# Returns all `Developer` nodes where the name property includes the letter `o`
developers = await Developer.find_many({"name": {"$contains": "o"}})

print(developers) # [<Developer>, <Developer>, ...]

# Or if no matches were found
print(developers) # []
```

##### Projections <a name="model-find-many-projections"></a>

`Projections` can be used to only return specific parts of the models as dictionaries. For more information about projections, see the [`Projections`](#query-projections) section.

```python
# Returns dictionaries with the developers name at the `dev_name` key instead
# of model instances
developers = await Developer.find_many({"name": "John"}, {"dev_name": "name"})

print(developers) # [{"dev_name": "John"}, {"dev_name": "John"}, ...]
```

##### Auto-fetching nodes <a name="model-find-many-auto-fetching-nodes"></a>

The `auto_fetch_nodes` and `auto_fetch_models` parameters can be used to automatically fetch all or selected nodes from defined relationship properties when running the `find_many()` query. For more about auto-fetching, see [`Auto-fetching relationship properties`](#query-auto-fetching).

```python
# Returns developer instances with `instance.<property>.nodes` properties already fetched
developers = await Developer.find_many({"name": "John"}, auto_fetch_nodes=True)

print(developers[0].coffee.nodes) # [<Coffee>, <Coffee>, ...]
print(developers[0].other_property.nodes) # [<OtherModel>, <OtherModel>, ...]

# Returns developer instances with only the `instance.coffee.nodes` property already fetched
developers = await Developer.find_many({"name": "John"}, auto_fetch_nodes=True, auto_fetch_models=[Coffee])

# Auto-fetch models can also be passed as strings
developers = await Developer.find_many({"name": "John"}, auto_fetch_nodes=True, auto_fetch_models=["Coffee"])

print(developers[0].coffee.nodes) # [<Coffee>, <Coffee>, ...]
print(developers[0].other_property.nodes) # []
```

> **Note**: The `auto_fetch_nodes` and `auto_fetch_models` parameters are only available for classes which inherit from the `NodeModel` class.

#### Model.update_one() <a name="model-update-one"></a>

The `update_one()` method finds the first matching graph entity and updates it with the provided properties. If no match was found, nothing is updated and `None` is returned. Properties provided in the update parameter, which have not been defined on the model, will be ignored.

This method takes two mandatory arguments:

- `update`: A dictionary containing the properties to update.
- `filters`: A dictionary containing the filters to use when searching for a match. For more about filters, see the [`Filtering queries`](#query-filters) section.

```python
# Updates the `age` property to `30` in the first encountered node where the name property equals `John`
# The `i_do_not_exist` property will be ignored since it has not been defined on the model
developer = await Developer.update_one({"age": 30, "i_do_not_exist": True}, {"name": "John"})

print(developer) # <Developer age=25>

# Or if no match was found
print(developer) # None
```

##### Returning the updated entity <a name="model-update-one-new"></a>

By default, the `update_one()` method returns the model instance before the update. If you want to return the updated model instance instead, you can do so by passing the `new` parameter to the method and setting it to `True`.

```python
# Updates the `age` property to `30` in the first encountered node where the name property equals `John`
# and returns the updated node
developer = await Developer.update_one({"age": 30}, {"name": "John"}, True)

print(developer) # <Developer age=30>
```

#### Model.update_many() <a name="model-update-many"></a>

The `update_many()` method finds all matching graph entity and updates them with the provided properties. If no match was found, nothing is updated and a `empty list` is returned. Properties provided in the update parameter, which have not been defined on the model, will be ignored.

This method takes one mandatory argument `update` which defines which properties to update with which values.

```python
# Updates the `age` property of all `Developer` nodes to 40
developers = await Developer.update_many({"age": 40})

print(developers) # [<Developer age=25>, <Developer age=23>, ...]

# Or if no matches were found
print(developers) # []
```

##### Filters <a name="model-update-many-filters"></a>

Optionally, a `filters` argument can be provided, which defines which entities to update. For more about filters, see the [`Filtering queries`](#query-filters) section.

```python
# Updates all `Developer` nodes where the age property is between `22` and `30`
# to `40`
developers = await Developer.update_many({"age": 40}, {"age": {"$gte": 22, "$lte": 30}})

print(developers) # [<Developer age=25>, <Developer age=23>, ...]
```

##### Returning the updated entity <a name="model-update-many-new"></a>

By default, the `update_many()` method returns the model instances before the update. If you want to return the updated model instances instead, you can do so by passing the `new` parameter to the method and setting it to `True`.

```python
# Updates all `Developer` nodes where the age property is between `22` and `30`
# to `40` and return the updated nodes
developers = await Developer.update_many({"age": 40}, {"age": {"$gte": 22, "$lte": 30}})

print(developers) # [<Developer age=40>, <Developer age=40>, ...]
```

#### Model.delete_one() <a name="model-delete-one"></a>

The `delete_one()` method finds the first matching graph entity and deletes it. Unlike others, this method returns the number of deleted entities instead of the deleted entity itself. If no match was found, nothing is deleted and `0` is returned.

This method takes one mandatory argument `filters` which defines which entity to delete. For more about filters, see the [`Filtering queries`](#query-filters) section.

```python
# Deletes the first `Developer` node where the name property equals `John`
count = await Developer.delete_one({"name": "John"})

print(count) # 1

# Or if no match was found
print(count) # 0
```

#### Model.delete_many() <a name="model-delete-many"></a>

The `delete_many()` method finds all matching graph entity and deletes them. Like the `delete_one()` method, this method returns the number of deleted entities instead of the deleted entity itself. If no match was found, nothing is deleted and `0` is returned.

```python
# Deletes all `Developer` nodes
count = await Developer.delete_many()

print(count) # However many nodes matched the filter

# Or if no match was found
print(count) # 0
```

###### Filters <a name="model-delete-many-filters"></a>

Optionally, a `filters` argument can be provided, which defines which entities to delete. For more about filters, see the [`Filtering queries`](#query-filters) section.

```python
# Deletes all `Developer` nodes where the age property is greater than `65`
count = await Developer.delete_many({"age": {"$gt": 65}})

print(count) # However many nodes matched the filter
```

#### Model.count() <a name="model-count"></a>

The `count()` method returns the total number of entities of this model in the graph.

```python
# Returns the total number of `Developer` nodes inside the database
count = await Developer.count()

print(count) # However many nodes matched the filter

# Or if no match was found
print(count) # 0
```

##### Filters <a name="model-count-filters"></a>

Optionally, a `filters` argument can be provided, which defines which entities to delete. For more about filters, see the [`Filtering queries`](#query-filters) section.

```python
# Counts all `Developer` nodes where the name property contains the letters `oH`
# The `i` in `icontains` means that the filter is case insensitive
count = await Developer.count({"name": {"$icontains": "oH"}})

print(count) # However many nodes matched the filter
```


#### Instance.export_model() <a name="instance-export-model"></a>

Export the model instance to a dictionary containing standard python types. Since this method uses `pydantic.BaseModel.json()` internally, all arguments it accepts are also accepted by this method.

```python
developer = await Developer(
  name="John",
  age=24
).create()

# Export the model instance to a dictionary
developer_dict = developer.export_model()

print(developer_dict) # {"element_id": "4:08f8a347-1856-487c-8705-26d2b4a69bb7:6", "name": "John", "age": 24}
```

##### Case conversion <a name="instance-export-model-case-conversion"></a>

By default, the `export_model()` will just convert the model instance to a dictionary. If you want to convert the keys of the dictionary to `camelCase` when exporting it, you can do so by passing the `convert_to_camel_case` argument to the method and setting it to `True`.

> **Note**: This argument is very opinionated and you will probably not need it in most cases.

```python
developer = await Developer(
  name="John",
  age=24,
  some_multi_level_property=True
).create()

# Export the model instance to a dictionary with camelCase keys
developer_dict = developer.export_model(convert_to_camel_case=True)

print(developer_dict) # {"elementId": "4:08f8a347-1856-487c-8705-26d2b4a69bb7:6", "name": "John", "age": 24, "someMultiLevelProperty": True}
```

#### Instance.import_model() <a name="instance-import-model"></a>

Since you can export a model instance to a dictionary, it is also possible to import a model instance from a dictionary. This can be useful if you want to import a model instance from JSON and use a method on it straight away.

> **Note**: Since this method converts the provided dictionary directly to a database model, the `element_id` key must be provided in the dictionary. This is because the library needs to know which entity the model belongs to when importing it. If you don't want to provide the `element_id` key, you can use pydantic's native methods for importing/exporting models and then call the `Model.find_one()` method.

```python
developer_dict = {
  "element_id": "4:08f8a347-1856-487c-8705-26d2b4a69bb7:6",
  "name": "John",
  "age": 24
}

# Import the model instance from the dictionary
# After this import, any method can be called on the instance
developer = await Developer.import_model(developer_dict)

print(developer) # <Developer element_id="4:08f8a347-1856-487c-8705-26d2b4a69bb7:6" name="John", age=24>
```

##### Case conversion <a name="instance-import-model-case-conversion"></a>

By default, the `import_model()` will just try to import the model. If you converted the keys of the dictionary to `camelCase` when exporting it or get them from a JSON response, you need to pass the `convert_to_snake_case` argument as `True` to let the library know about it.

```python
developer_dict = {
  "element_id": "4:08f8a347-1856-487c-8705-26d2b4a69bb7:6",
  "name": "John",
  "age": 24,
  "someMultiLevelProperty": True
}

# Import the model instance from the dictionary
# After this import, any method can be called on the instance
developer = await Developer.import_model(developer_dict, convert_to_snake_case=True)

print(developer) # <Developer element_id="4:08f8a347-1856-487c-8705-26d2b4a69bb7:6" name="John", age=24, some_multi_level_property=True>
```

#### Model.register_pre_hooks() <a name="model-register-pre-hooks"></a>

Allows you to manually define pre-hooks for methods like described in the [`Defining settings for a model`](#defining-settings-for-a-model) section.

```python
def print_name(self, *args, **kwargs):
  print(self.name)

# Registers the `print_name` method as a pre-hook for the `update_one` method
Developer.register_pre_hooks({"update_one": print_name})
```

##### Overwriting existing hooks <a name="model-register-pre-hooks-overwrite"></a>

You can choose to overwrite all hooks previously defined for the model by providing the `overwrite` parameter as `True`. This will clear all previously defined hooks, regardless of whether they were defined using this method or model settings.

#### Model.register_post_hooks() <a name="model-register-post-hooks"></a>

Same behavior as [`Model.register_pre_hooks()`](#model-register-pre-hooks), but the hooks are registered as `post-hooks`.

#### Model.model_settings() <a name="model-model-settings"></a>

Returns the settings defined for the current model.

```python
model_settings = Developer.model_settings()

print(model_settings) # <NodeModelSettings labels={"Developer", "Python"}, auto_fetch_nodes=False, ...>
```
