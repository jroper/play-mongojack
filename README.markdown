MongoJack Play 2.1 Module
=========================

This project provides a very simple plugin for Play 2.1 that allows easy access of MongoJack wrapped connections to MongoDB.  [MongoJack](http://mongojack.org) is a lightweight POJO mapper that uses Jackson to serialise/deserialise MongoDB documents.  Because it uses Jackson, with bson4jackson to parse responses, it is fast, very flexible and performant.  It provides most of the same CRUD methods that the MongoDB Java driver provides, plus a more convenient query and updating interface.

Mailing list
------------

For any questions, please use the [MongoJack users mailing list](http://groups.google.com/group/mongo-jackson-mapper).

Installation
------------

Add the following dependency to your dependencies:

    "org.mongojack" %% "play-mongojack" % "2.0.0-RC2" 

for example, in ``Build.scala``:

    val appDependencies = Seq(
        "org.mongojack" %% "play-mongojack" % "2.0.0-RC2" 
    )

Scala quick start
-----------------

    import com.fasterxml.jackson.annotate.JsonProperty
    import reflect.BeanProperty
    import javax.persistence.Id
    import play.api.Play.current
    import play.modules.mongojack.MongoDB
    import scala.collection.JavaConversions._

    class BlogPost(@ObjectId @Id val id: String,
                 @BeanProperty @JsonProperty("date") val date: Date,
                 @BeanProperty @JsonProperty("title") val title: String,
                 @BeanProperty @JsonProperty("author") val author: String,
                 @BeanProperty @JsonProperty("content") val content: String,
                 @BeanProperty @JsonProperty("comments") val comments: List[Comment]) {
        @ObjectId @Id def getId = id;
    }

    object BlogPost {
        private lazy val db = MongoDB.collection("blogposts", classOf[BlogPost], classOf[String])

        def save(blogPost: BlogPost) { db.save(blogPost) }
        def findById(id: String) = Option(db.findOneById(id))
        def findByAuthor(author: String) = db.find().is("author", author).asScala
    }

A few notes:

* ``MongoDB.collection`` has an implicit ``Application`` argument.  The easiest way to ensure you have an implicit ``Application`` available is to import ``play.api.Play.current``.  Alternatively, use ``getCollection``, and the current application will automatically be used.
* MongoDB requires ids to have the name ``_id``, however, Scala won't let you annotate a field that starts with an underscore as ``@BeanProperty``.  The simplest solution is to use the ``@Id`` annotation, but then you also need to provide your own getter, otherwise when you serialise it, Jackson won't know that the ``id`` field should be serialised as ``@Id``.
* The reason each property needs to be ``@JsonProperty`` annotated is that the JVM doesn't allow putting method parameter names into bytecode, so Jackson can't reflect on the constructor to find out which argument is for which field.
* Queries are returned as Java iterables, so you probably want to convert them to Scala iterables.
* Returning single results as an ``Option`` allows more idiomatic use of the result.

Configuration
-------------

    # Configure the database name
    mongodb.database=databasename
    # Configure credentials
    mongodb.credentials="user:pass"
    # Configure the servers
    mongodb.servers=host1.example.com:27017,host2.example.com,host3.example.com:19999

The database name defaults to play.  The servers defaults to localhost.  Specifying a port number is optional, it defaults to the default MongoDB port.  If you specify one server, MongoDB will be used as a single server, if you specify multiple, it will be used as a replica set.

Configuring the object mapper
-----------------------------

If you need to adjust Jackson's object mapper, you need to provide a class as a play-plugin that implements the trait ``ObjectMapperConfigurer``. This trait has two methods, one for configuring the global object mapper that will be used for all collections, and another for configuring object mappers per collection.  An example implementation might look like this:

    class MyObjectMapperConfigurer(app: Application) extends Plugin with ObjectMapperConfigurer {
        def configure(defaultMapper: ObjectMapper) =
            defaultMapper.configure(DeserializationConfig.Feature.FAIL_ON_UNKNOWN_PROPERTIES, false)

        def configure(globalMapper: ObjectMapper, collectionName: String, objectType: Class[_], keyType: Class[_]) = {
            if (collectionName == "something") {
                // Because object mapper is mutable, and doesn't provide a simple way to just copy it's configuration, if
                // you want to configure one for a specific collection, you probably have to create a new one from scratch.
                val mapper = configure(MongoJacksonMapperModule.configure(new ObjectMapper).withModule(new DefaultScalaModule))
                return mapper.configure(DeserializationConfig.Feature.ACCEPT_SINGLE_VALUE_AS_ARRAY, true)
            }
            return globalMapper
        }
    }
	
To activate the configurer you need to add it to your ``conf/play.plugins`` file. An example might look like this:

	1001:foo.bar.MyObjectMapperConfigurer
	

Documentation
-------------

For documentation on how to use MongoJack itself, please visit the [MongoJack site](http://mongojack.org).  In particular, the [tutorial](http://mongojack.org/tutorial.html) might be a good place to start.

Features
--------

* Manages lifecycle of MongoDB connection pool
* Caches JacksonDBCollection instances, so looking up a JacksonDBCollection is cheap
* Configures Jackson to use the FasterXML ``DefaultScalaModule``, so scala mapping works out of the box.
* Allows configuring a custom object mapper

