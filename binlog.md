# EpiK Binlog Spec

## Log format

Basically every Log is composed of 2 components: 

<img src="https://github.com/EpiK-Protocol/epik-gateway-binlog-encoder/blob/master/images/log_format.png" width="400px">

1. _Mode_ , a bitMap, used to interpret the _Payload_, different bit combination indicate different log type.
2. _Payload_, the actual log content.

_Mode_ is 1 byte long, which is divided into 4 subsections from lowest to the highest bit.
1. The lowest 2 bits are Add/Update/Remove mode.
2. The next 2 bits are GraphDB/Tag/Edge mode.
3. The next 1 bit is Schema/Data(instances) mode.
4. The highest 3 bits are reserved for later usage.

<img src="https://github.com/EpiK-Protocol/epik-gateway-binlog-encoder/blob/master/images/bitmap_mode.png" width="500px">

Combination of those bits, can form any of the following logs.
1. GraphDB/Tag/Edge's Schema Add/Update/Remove Log.
2. Tag/Edge's Data(instances) Add/Update/Remove Log.


### GraphDB Schema Log
GraphDB Schema Log is used to manipulate graph space's defination in schema.
![schema_db](images/schema_db.png)

In order to support multi knowledge graphs, GraphDB is introduced to support conceptual and physical data organization, which has a counterpart in Relational Database Management System(RDBMS).
GraphDB's naming convention follows the identifier naming convention of typical programming language such as c/c++, which should be composed of alphabet、digit、underscore and $, and should not starts with 
digit. GraphDB names that starts with underscore or $ are reserved for internal usage. Since GraphDB's name can be of variable length, name's size should be encoded. 
We use a [Nebula Graph](https://nebula-graph.io/) as our backend graph database. Nebula Graph is a distributed graph database, __partition number__ and __replication factor__ should be specified when creating a graphspace.
which could not be altered after creation., so in order to harvest the power when scale-out, a modest partition number should be used, 1024 or higher is recommended.
Replication is used to maintain fault-tolerance. Graph database should be functional under at most minority of all replicas fails.
But replication has its drawback too, it will cause storage-bloat when set too high, an odd small number such as 3、5 or 7 is recommended, the most common setting is 3.


### Tag Schema Log
Tag Schema Log is used to manipulate Tag's defination in schema.
![schema_tag](images/schema_tag.png)
In order to support multi-graphdb, __GraphDB Name__ and __GraphDB Name Size__ is added as prefix to indicate which graphdb this Tag Schema Log belongs to.
Although usually graph database maintain a internal GraphDB id to uniquely identify it, it also maintains the relationship between GraphDB Name and GraphDB id.
But we could not know the GraphDB id before it actually created in graph database, and got assigned its GraphDB id, we should make our log self-contained.
so we put <GraphDB Name Size，GraphDB Name>, a tuple2 composite field in every log to distinguish which graphdb it belongs to.

Tag is a label which assign to a Vertex, which may usually represent a `concept` a vertex has, is a logical group of properties.
And Vertex is conceptually a tuple2 of <VID,TagList>, which VID is a integer assigned by Nebula Graph to uniquely identify a vertex internally, and TagList is a collection of Tags, which indicates that a vertex can have many concepts.


A Tag is conceptually a tuple2 of <TagName, PropertiesList>, which TagName is Tag' name(variable length, so its name's size should be encoded inside), and PropertiesList is a collection of Property, which defined as:
1.	Property Name Size: length of property name.
2.	Property Name: name of the property.
3.	Property Type：support common data type, including: integer、double、string and timestamp. When of type string, since it is variable length, an extra size field will be encoded to mark its boundary.
4.  HasDefaultValue: whether has default value. when in schema definition and HasDefaultValue=1, then the following 2 fields: __Property Default Value Size__ and __Property Default Value__ must be provided. when in data definition, HasDefaultValue is meaningless, keeping it is just to keep format consistent,
5.	Property [Default/Actual] Value Size: value of the property according to Property Type.
6.	Property [Default/Actual] Value: value of the property according to Property Type.
In order to support __PropertiesList__, an extras __Properties Szie__ field to count how many properties a Tag has.


### Edge Schema Log
Edge Schema Log is used to manipulate Edge's defination in schema. 
![schema_edge](images/schema_edge.png)
It bears much resemblance with Tag Schema Log. The only different is that we change __Tag Name Size__ to __Edge Name Size__, and change __Tag Name__ to __Edge Name__.

###	Vertex Data Log
Vertex Data Log is used to manipulate vertex's data.
![data_vertex](images/data_vertex.png)
As mentioned in __Tag Schema Log__，Vertex is conceptually a tuple2 of <VID, TagList>.
Nebula Graph make a conversion from domain values to integer vertex(VID) id to reduce to maintenance overhead. Nebula Graph also expose native functions to do for application developments.
such as __Hash__ or __Fixed-size String__. We use __Hash__ function, but it is our responsibility to make sure that every domain entity(business entity) is unique.
Vertex Data Log is composed of the following:
1. Vertex Domain Value Size: value's size of the domain entity,to mark its boundary.
2. Vertex Domain Value： domain entity's value，must be of string type.
3. VID Generator Type: the function we used to make the domain entity to VID conversion. we use __Hash__.
4. TagList: a collection of Tag，a Tag is tuple6 of<Property Name Size, Property Name, Property Type, HasDefaultValue, Property Value Size, Property Value>, which every __Size__ field is used to mark the boundary of the corresponding __Value__ field. 

### Edge Data Log
__Edge Data Log__ is used to manipulate edge's data.
![data_edge](images/data_edge.png)
It bears much resemblance with __Vertex Data Log__ with minor differences:
1.	Edge is connected by a __From Vertex__ and a __To Vertex__, which require __From Vertex Domain Value__ and __To Vertex Domain Value__，and corresponding Value Sizes fields.
2.	Edge does not support multi-Tag，so does not need introduce filed designed for multi-Tags，Edge's properties just lay one after another，every property is tuple5 of <Property Name Size, Property Name, Property Type, Property Value Size, Property Value>.
