# Rell

Advantages of Rell
Simplicity: Rell syntax is clean and easy to understand. You can define relationships without getting bogged down by the complexities of foreign keys and constraints.
Readability: Rell code is more readable and maintainable, making it easier to understand the structure of your database at a glance.
Error reduction: By abstracting away the manual management of keys and constraints, Rell reduces the potential for errors and inconsistencies in your database design.

## SQL table vs Rell entity
https://learn.chromia.com/courses/rell-masterclass/entities

We start by looking at how to define an entity and what the corresponding SQL table will look like. An entity is represented by a SQL table and is defined using the keyword entity. Each attribute represents a column of the SQL table.

```rell
entity my_entity {
    key my_key: text;
    index my_index: text;
    mutable my_mutable: text;
    my_immutable: text;
}
```

It's possible to add the keywords key, index, or mutable to each attribute. The key annotation adds a unique index on the column, while index adds a non-unique index. The mutable annotation informs Rell that the property can be changed, but it does not directly affect the database. Let's examine what an entity looks like in the database.

```rell
entity my_entity {
    key my_key: text;
    index my_index: text;
    mutable my_mutable: text;
    my_immutable: text;
}
```

```sql
create table "c0.my_entity"(
  "rowid" bigint not null,
  "my_key" text not null,
  "my_index" text not null,
  "my_mutable" text not null,
  "my_immutable" text not null,
  constraint "PK_c0.my_entity"
  primary key ("rowid"),
  constraint "K_c0.my_entity_0"
  unique ("my_key")
);
create index "IDX_c0.my_entity_0" on "c0.my_entity"("my_index");
```

First, we observe that a column rowid has been added as the primary key, which Rell uses to reference each entity. Next, my_key receives a unique constraint, implicitly indicating that it has a UNIQUE INDEX in the database. The attribute my_index becomes a column with an index. Both the mutable and immutable columns are equal in the database.


### Foreign keys

In a relational database, having several tables and references between them is common. This is enforced by adding foreign keys to a column. In Rell, this is done by referencing another entity as an attribute. Let's consider a simple example: one table representing a street address and one representing a house.

```rell
entity street {
  key address: text;
}

entity house {
  index street;
  number: integer;
  number_of_rooms: integer;
  number_of_floors: integer;
  floor_area: integer;
}
```

```sql
create table "c0.street"(
  "rowid" bigint not null,
  "address" text not null,
  constraint "PK_c0.street"
  primary key ("rowid"),
  constraint "K_c0.street_0"
  unique ("address")
);
create table "c0.house"(
  "rowid" bigint not null,
  "street" bigint not null,
  "number" bigint not null,
  "number_of_rooms" bigint not null,
  "number_of_floors" bigint not null,
  "floor_area" bigint not null,
  constraint "PK_c0.house"
  primary key ("rowid"),
  constraint "c0.house_street_FK"
  foreign key ("street")
  references "c0.street" ("rowid")
);
create index "IDX_c0.house_0" on "c0.house"("street");
```

For each street address, we can have several houses. As seen here, referencing the street entity from the house entity corresponds to a bigint value in the house table, with a foreign key constraint on the street table rowid. When declaring the foreign key, it's also possible to set another name, specifying the reference as type: index my_street: street;.

It's also possible to add composite indexes. By specifying key attribute1, attribute2; or index attribute1, attribute2;, we can index based on multiple columns, depending on whether this combination must be unique. For example, we know that there can only be one house with a specific number, so we make the following change to the entity:

```rell
entity house {
  index street;
  number: integer;
  key street, number;
  number_of_rooms: integer;
  number_of_floors: integer;
  floor_area: integer;
}
```

This will correspond to the following SQL query:
```sql
create table "c0.house"(
  "rowid" bigint not null,
  "street" bigint not null,
  "number" bigint not null,
  "number_of_rooms" bigint not null,
  "number_of_floors" bigint not null,
  "floor_area" bigint not null,
  constraint "PK_c0.house"
  primary key ("rowid"),
  constraint "c0.house_street_FK"
  foreign key ("street")
  references "c0.street" ("rowid"),
  constraint "K_c0.house_0"
  unique (
      "street",
      "number"
    )
);
create index "IDX_c0.house_0" on "c0.house"("street");
```

A unique constraint was added with a combination of street and number columns.

For a deeper understanding of managing relationships between entities, check out our Understand relationships in Rell course.


> Tip: Define only a few entities. A complex database model will likely result in many joins, making the dapp slower.

### Practical Example

```rell
entity street {
  key address: text;
}

entity house {
  index street;
  number: integer;
  key street, number;
  number_of_rooms: integer;
  number_of_floors: integer;
  floor_area: integer;
}

operation create_street(address: text) {
    create street( address );
}

query get_all_streets() = street @* { } ( id = .rowid, .address );

operation create_house(street_id: rowid, number: integer, number_of_rooms: integer, number_of_floors: integer, floor_area: integer) {
    val street = street @ { .rowid == street_id };
    create house( street, number, number_of_rooms, number_of_floors, floor_area );
}

query get_all_houses() = house @* { } ( $.to_struct() );
```

## SELECT statement

The most common way to select a record from the database is by using the @ at-operator.

```
FROM        CARDINALITY  WHERE         WHAT  TAIL
entity_name @            { condition } ()    ;
```

The operator is separated into five parts: FROM, CARDINALITY, WHERE, WHAT, and TAIL.

FROM: represents the table from where to make the query. Same as SQL FROM.
CARDINALITY: specifies the number of results that are expected from the query. The query will fail if this condition can't be matched.
@ exactly one result
@? zero or one result
@+ more than one result
@* zero or many results
WHERE: represents a filter similar to the WHERE part of an SQL query.
WHAT: The actual values to retrieve from the query. It can be one or several attributes or left empty to get a reference (rowid) of the entity. Here, you can also   sort, group, or omit fields.
TAIL: Additional SQL-related statements such as limit or offset that can be used when retrieving multiple results.
tip
The at-operator also works on collection types such as list<T> and set<T> as a powerful replacement for the for loop.

Implicit select statements
There are also cases where SELECTs are performed without the reader noticing. This happens when we access an attribute of an entity.

```rell
val attribute = entity_instance.attribute;
```

This will result in the following SQL query:

```sql
SELECT "attribute" FROM "entity_name" WHERE "rowid" = ? ORDER BY "rowid"
```

To make the database roundtrip explicit, we can form the following at-expression:

```rell
val attribute = entity_name @ { $ == entity_instance } ( .attribute );
```

The expression provided above will correspond to an equivalent SQL expression.

You can use the .to_struct() function to retrieve all attributes from an entity. This will give you an in-memory representation of the entity. However, it is important to note that this will also create a database roundtrip to retrieve the values. This means that the following statements are equivalent:

```rell
val struct1 = entity_instance.to_struct();
val struct2 = entity_name @ { $ == entity_instance } (struct<entity_name>(attribute = .attribute, ...));
```

Select statement example
Based on the above, we see that excessive use of attribute access can lead to many SELECTs. Have a look at the following function:

```rell
function is_mansion(house) {
  if (house.floor_area <= 460) return false;
  if (house.number_of_rooms < 10) return false;
  return house.number_of_floors > 1 and house.number_of_floors <= 3;
}
```

This may seem like a reasonable function, but in reality, this will perform four database roundtrips, getting the attributes one by one. Putting this in a loop makes the situation even worse:

```rell
val houses = house @* {};
for (house in houses) {
  if (is_mansion(house)) print("Found a mansion");
}
```

The number of database roundtrips will increase linearly with the size of the database table.

> danger: This means that passing entity references to functions as arguments is really dangerous as it can lead to a lot of unnecessary database roundtrips.

Consider the difference when we change the signature to function is_mansion(house: struct<house>). With this change, we can retrieve all the data we need in a single database statement:

```rell
val houses = house @* {} ($.to_struct());
for (house in houses) {
  if (is_mansion(house)) print("Found a mansion");
}
```

> tip: Don't pass entity references in functions, as it might lead to additional database roundtrips that can grow uncontrollably in the worst case.

### Practical example
Add the following query to the queries.rell file stored in the src/main directory.

This query retrieves all house records:

```rell
query get_all_mansion() = house @* { .floor_area >= 406, .number_of_rooms > 10, .number_of_floors > 1 } ( $.to_struct() );
```

## INNER JOIN statement

Joining two tables is simple in Rell. We do this by putting all the tables we want to join in the FROM part of the at-expression and specify the constraint between them in the WHERE part:

```rell
function join_example() {
    val res = (house, street) @* { street == house.street };
}
```

The Rell code above will correspond to the following SQL query:


```sql
SELECT A00."rowid", A01."rowid"
    FROM "street" A00, "house" A01
    INNER JOIN "street" A02 ON A01."street" = A02."rowid"
    WHERE A00."rowid" = A02."rowid"
    ORDER BY A00."rowid", A01."rowid"
```

### Implicit statements
Now, in this scenario, we have a foreign key constraint on street from house, so we can access street directly from an instance of house, for example:

```rell
val street = house_instance.street;
```

This will also correspond to a join statement:

```sql
SELECT A01."rowid"
    FROM "house" A00
    INNER JOIN "street" A01 ON A00."street" = A01."rowid"
    WHERE A00."rowid" = ?
    ORDER BY A00."rowid"
```

Similarly, accessing street in any way when working with house will result in a join.

```rell
house @ { .street.address == "FakeStreet" };
house @ {} (.street);
house_instance.to_struct();
```

Even though joining a table with a foreign key constraint may not be a heavy join, it is important to remember which Rell statements will create a join. For example, we used the struct<house> construct in the previous example in the function expression. If house has several references to other tables, the to_struct() call will make several joins. For example, let's add some references to the previous example:

```rell
entity garage {}
entity house_key {}
entity house {
    ...
    garage;
    house_key;
}
```

### Avoiding unnecessary joins
If we, in this case, use the same construct as before:

```rell
val houses = house @* {} ($.to_struct());
```

This joins both garage and house_key entities. If these are not needed in the function, we should be more explicit in the at-expression:

```rell
struct house_info {
    number_of_rooms: integer;
    number_of_floors: integer;
    floor_area: integer;
}
val houses = house @* { } ( house_info( .number_of_rooms, .number_of_floors, .floor_area ) );
```

By changing the function's signature to is_mansion(house: house_info), we can optimize which values we get from the database.

### Practical example

In this example we use a set of queries to retrieve filtered records based on the query specifics.

```rell
query get_houses_with_streets() = (house, street) @* { street == house.street } ( street = .street.address, .number );
query get_houses_with_specific_street(street_address: text) = house @* { .street.address == street_address } ( .number, street = .street.address );
query get_houses_info() = house @* { } ( house_info( .number_of_rooms, .number_of_floors, .floor_area ) );
```

## Code optimization example

Let's try what we learned in a concrete example. We define a small app that holds a set of NFTs. Each NFT has a type and some underlying data. An NFT can be owned by a single user. We can model this in the following way.


```rell
entity nft {
  key id: integer;
  index type: nft_type;
  data: byte_array;
  mutable expired: boolean = false;
}

enum nft_type { A,B }

struct nft_a {
  condition: boolean;
  data: integer;
}

struct nft_b {
  data: byte_array;
}

entity nft_owner {
  key nft;
  index owner: user;
}

entity user {
  key name;
}
```

Each NFT gets a unique ID and a type. Each type corresponds to a structure nft_a or nft_b with different underlying data structures. Each user is required to have a unique name, and the nft_owner that holds the two entities together has a unique constraint on the NFT and is a joint constraint with the NFT and the user so that a user can own several NFTs, but an NFT can only have a single owner.

Let's now say that we want to create a query that gets all NFTs of type A where the condition holds true for a particular user. A very tempting way to write such a query could be:

```rell
query get_valid_a(user_name: name): list<(id: integer, data: integer)> {
  val user = user @ { user_name };
  val nfts = nft_owner @* { user }.nft;
  return nfts @* { is_valid_a($) } (.id, data = nft_a.from_bytes($.data).data);
}
```

```rell
function is_valid_a(nft): boolean {
  if (nft.type != nft_type.A) return false;
  if (nft.expired) return false;
  val data = nft_a.from_bytes(nft.data);
  return data.condition;
}
```

The query fetches a user and then all its NFTs. For each NFT, we check if it is valid and then return the results as a list of named tuples linking the IDs with the data. The conditions are checked in a separate function to reduce bloat.

To investigate the database footprint of this query, we define a few operations to help us:

```rell
operation create_user(name) {
  create user(name);
}

operation create_nft(id: integer, type: nft_type, data: byte_array, owner_name: name) {
  val owner = user @ { owner_name };
  when (type) {
    nft_type.A -> nft_a.from_bytes(data);
    nft_type.B -> nft_b.from_bytes(data);
  }
  val nft = create nft(id, type, data);
  create nft_owner(nft, owner);
}
```

We then create a test case where we create a user and 10 NFTs before doing the query:

```rell
@test module;

import ^^.main.{ create_user, create_nft, nft_type, nft_a, get_valid_a };

function test_nft() {
    print("Creating user");
    rell.test.tx().op(create_user("Alice")).run();
    print("Creating nfts");
    for (id in range(10)) {
        rell.test.tx().op(create_nft(id, nft_type.A, nft_a(true, id*10).to_bytes() ,"Alice")).run();
    }
    print("Perform query");
    get_valid_a("Alice");
}
```

Looking at the output after the line Perform query in the output, we see:

```sql
SELECT A00."rowid" FROM "c0.user" A00 WHERE A00."name" = ? ORDER BY A00."rowid"
SELECT A02."rowid" FROM "c0.nft_owner" A00 INNER JOIN "c0.user" A01 ON A00."owner" = A01."rowid" INNER JOIN "c0.nft" A02 ON A00."nft" = A02."rowid" WHERE A01."rowid" = ? ORDER BY A00."rowid"
SELECT A00."type" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."expired" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."type" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."expired" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."type" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."expired" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."type" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."expired" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."type" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."expired" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."type" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."expired" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."type" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."expired" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."type" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."expired" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."type" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."expired" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."type" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
SELECT A00."expired" FROM "c0.nft" A00 WHERE A00."rowid" = ? ORDER BY A00."rowid"
```

This query made 22 database roundtrips. It first makes a query to get the rowid of the user followed by a really long query which joins the three tables. After this, two lines gets repeated over and over again. This means that this query will scale proportionally in number of database requests with the number of nfts for a user.

If we instead join all tables to start with, and select all fields from the nft directly, we could optimize this query to make a single database query. Here is an updated version of the query:

```rell
query get_valid_a(user_name: name): list<(id: integer, data: integer)> {
  val nfts = (user, nft_owner) @* { // <-- 1.1 Join the tables directly
    user.name == user_name,         // <-- 2. Criteria sorting, most distinct condition first
    user == nft_owner.owner         // <-- 1.2 Join cont.
  } (
    id = nft_owner.nft.id,
    nft = nft_owner.nft.to_struct() // <-- 3. select all fields from the nft (be more specific if needed)
    );
  return nfts @* { is_valid_a(.nft) } (.id, data = nft_a.from_bytes(.nft.data).data);
}

function is_valid_a(nft: struct<nft>): boolean { // <-- 4. Takes a struct as argument
  if (nft.type != nft_type.A) return false;
  if (nft.expired) return false;
  val data = nft_a.from_bytes(nft.data);
  return data.condition;
}
```

Let's see what we did:

We joined the tables user and nft_owner because the user will return a single value at most; this is fine.
We put the most specific criteria first.
We used the to_struct() to select all attributes directly.
We ensured that our function takes an in-memory version of the NFT to avoid making separate selects in the function.
Rerunning the test produces the following result:

```rell
SELECT A03."id", A03."id", A03."type", A03."data", A03."expired"
    FROM "c0.user" A00, "c0.nft_owner" A01
    INNER JOIN "c0.user" A02 ON A01."owner" = A02."rowid"
    INNER JOIN "c0.nft" A03 ON A01."nft" = A03."rowid"
    WHERE (A00."name" = ?) AND (A00."rowid" = A02."rowid")
    ORDER BY A00."rowid", A01."rowid"
```

We successfully improved our query to make a single database roundtrip regardless of the number of NFTs in the table.

## UPDATE statement

It is only possible to update the value of an attribute that has been explicitly marked with mutable:

```rell
entity my_entity {
    key my_key: text;
    mutable my_mutable: text;
}
```

If you have a reference to an entity, updating an attribute is as simple as making an assignment:

```rell
my_entity_instance.my_mutable = "new value";

```

This corresponds to the following SQL statement:

```sql
UPDATE "c0.my_entity" A00 SET "my_mutable" = ? WHERE A00."rowid" = ?
```

If you have multiple mutable attributes to update, assigning them one by one may be tempting, causing a new database roundtrip each time. A better approach is to use the update keyword in Rell. It works similar to the at-operator.

```rell
FROM        CARDINALITY  WHERE         WHAT
update entity_name @            { condition } ( .my_mutable = "new value" );
```

The WHAT part has been replaced with an assignment, and the TAIL part is omitted.

This way, we can update several fields in a single database roundtrip. If we already have a reference to an entity, we can pass it as the condition $ == entity_instance.


### Example

```rell
entity user_info {
    key name;
    key pubkey;
    mutable address: text;
}

operation update_address(name, new_address: text) {
    val user = find_and_check_signer(name);
    user.address = new_address;
}

function find_and_check_signer(name): user_info {
    val user_pubkey = user_info @ { name } (user = $, pubkey = .pubkey);
    require(op_context.is_signer(user_pubkey.pubkey), "User must sign operation");
    return user_pubkey.user;
}
```

Here, we have a user entity with a unique key in its name, a public key, and a mutable address property. In operation update_address, we want to find the user and update the address. The function find_and_check_signer thus retrieves not only the user entity with a specific name in the first at-expression. It also fetches the pubkey property to be able to require a signature matching this key without making an additional SQL select. The new address is then updated with a simple assignment.

Let's say that we instead want the address split into a street name and a zip code. The new model would thus look like:

```rell
entity user_info {
    key name;
    key pubkey;
    mutable street_address: text;
    mutable zip_code: integer;
}
```

We would then update our operation to:

```rell
operation update_address(name, new_street_address: text, new_zip_code: integer) {
    val verified_user = find_and_check_signer(name);
    update user_info @ { $ == verified_user } (
       .street_address = new_street_address,
       .zip_code = new_zip_code
    );
}
```

### Practical example

```rell

entity user_info {
    key name;
    key pubkey;
    mutable street_address: text;
    mutable zip_code: integer;
}

function find_and_check_signer(name): user_info {
    val user_pubkey = user_info @ { name } (user = $, pubkey = .pubkey);
    require(op_context.is_signer(user_pubkey.pubkey), "User must sign operation");
    return user_pubkey.user;
}

operation create_user_info(name: text, pubkey: byte_array, street_address: text, zip_code: integer) {
    create user_info( name, pubkey, street_address, zip_code );
}

operation update_address(name, new_street_address: text, new_zip_code: integer) {
    val verified_user = find_and_check_signer(name);
    update user_info @ { $ == verified_user } (
       .street_address = new_street_address,
       .zip_code = new_zip_code
    );
}

query get_users_info() = user_info @* { } ( $.to_struct() );
```

## Keys and indexes

Indexes are crucial in improving database performance by allowing faster data retrieval. To understand the significance of indexes in Rell, let's start with the basics.

Imagine that you have a large dataset and need to find a specific entry within it. Without an index, searching through all the data can be time-consuming with a worst-case time complexity of O(n). However, when an attribute is indexed in Rell, it dramatically enhances query performance, reducing the time complexity to O(log(n)), thanks to PostgreSQL's underlying binary tree structure.

In Rell, you have two options to mark an attribute for indexing: key and index. While both improve query performance, they serve different purposes:

Key
A key is an index that enforces a unique constraint on the indexed attribute. This uniqueness constraint guarantees that entities with the same attribute value will not exist in your database, making it less error-prone when creating entries.

Index
An index is used to improve query performance for non-unique attributes.

### Key and index example
Consider the scenario where multiple houses share the same street, each with its unique id. This could be modelled like:


```rell
entity house {
    key id: integer;
    index street: text;
}
val unique_house = house @ { .id == 10 };
val houses_on_main_street = house @* { .street == "main street" };
```

### Composite index
To further optimize your database queries, you can add a composite index. This is particularly useful when finding entries based on a combination of attributes. For instance, if your app often queries all houses owned by a specific owner on a particular street, you can create a multi-column index.

```rell
index owner, street;
```

The order of the fields in a multi-column index is crucial for performance. This is how composite indexes work in SQL, where each column precedes the latter. To create the most efficient index, place the most significant attribute first. For instance, if you want to query all houses with exactly four rooms and a floor area greater than 100:

```rell
house @ { .number_of_rooms == 4, .floor_area > 100 };
```

In this case, an optimal index configuration would be:
```rell
index number_of_rooms, floor_area;
```

Composite keys work the same way. If a specific combination of entries can only exist once, adding a composite key will ensure this constraint in the database. Place the attribute yielding the fewest results first to achieve optimal performance.

### Practical example
This example is focused on creating the house records and requesting specific house records.

```rell
entity house {
  index street;
  number: integer;
  key street, number;
  number_of_rooms: integer;
  number_of_floors: integer;
  floor_area: integer;
  index number_of_rooms, floor_area;
}

query get_specific_houses(number_of_rooms: integer, floor_area: integer) = house @* { .number_of_rooms == number_of_rooms, .floor_area >= floor_area } ( $.to_struct() );
```

## Subqueries

In this lesson, we will explore how to use nested expressions in Rell to create efficient database queries. We will learn how to use nested expressions in the exists() and empty() functions, as well as the in operator. Nested expressions allow you to create more complex and efficient database queries, significantly improving the performance and functionality of your Rell queries.

Subqueries are used to perform a query within another query to refine search results. For example, if you want to find customers who have placed orders totaling more than $500, you might first use a subquery to calculate the total spent by each customer. This subquery returns a specific result, such as an array of objects containing customer details.

Then, the main query will use this information to list only those customers who meet the criteria. So, the subquery identifies customers with high spending, and the main query retrieves their details.

Adding new records
In order to illustrate the subquery mechanism, we must create two entities, customer and order, in the src/main/entities.rell file. In this example, we will use these entities to:

Find all customers who placed at least one order.

Find customers who have not yet placed any orders.

Find customers who have purchased a specific product.

Below is the code for creating the customer and order entities:

```rell
entity customer {
  name: text;
  city: text;
  email: text;
}

entity order {
  order_id: text;
  customer_email: text;
  product: text;
}

operation create_customer(name: text, city: text, email: text) {
    create customer( name, city, email );
}

operation create_order(order_id: text, customer_email: text, product: text) {
    create order( order_id, customer_email, product );
}
```

### Running subqueries

The Rell language includes some built-in global functions in the System Library that can help create a subquery. There are two functions and an operator for performing a subquery:

exists() - This function checks if a value exists. It returns true if a value exists; otherwise, it returns false. It expects a value or a collection of values.
empty() - This function identifies if a value is null or an empty collection. It returns true if a value/collection is null; otherwise, it returns false.
in - This operator verifies if a specific element exists in a collection. It returns true if a value exists; otherwise, it returns false.
Example of using exists()

### Example of using exists()
The following subquery finds all customers who have placed at least one order. In this example, the exists function is invoked as a subquery, which returns a boolean value indicating the presence or absence of an element or a collection of elements. If it returns true, the main query is invoked, returning a collection of customers.

If it returns true, the main query is invoked, returning a collection of customers.

To implement this example, add the following code to the src/main/queries.rell file:

```rell
query get_customers_with_orders() {
    return customer @* {
        exists(order @* { .customer_email == customer.email })
    } ( $.to_struct() );
}
```

### Example of using empty()
```rell
query get_customers_without_orders() {
    return customer @* {
        empty(order @* { .customer_email == customer.email })
    } ( $.to_struct() );
}
``` 

### Example of using in
```rell
query get_customers_who_ordered_phone() {
    return customer @* {
        .email in order @* { .product == "phone" } ( order.customer_email )
    } ( $.to_struct() );
}
```

## Understand relationships in Rell
 we will explore data modelling and creating relationships between entities using the Rell language. Our goal is to help you understand table relationships and use them to build efficient and scalable applications.

We assume you have a basic understanding of SQL, including table relationships and joins. We will show you how to implement these concepts using Rell, a powerful language for managing data structures. You will learn to translate your SQL knowledge into Rell and create robust and scalable data models.

By the end of this course, you will have a solid understanding of data modelling with Rell, enabling you to build advanced applications with efficient data architecture. Get ready to enhance your skills and take your applications to the next level!

What we will learn
In this course, we will explore how to implement different types of relationships in Rell:

One-to-one
One-to-many
Many-to-many
We will discuss creating and using these relationships to build functional applications. Additionally, we will cover the following types of joins:

Inner Join
Outer Join
Concept and structure of our application
We will create a library management application that will include the following main entities:

Library: Information about the library, its address, and shelves.
Shelf: The number of the shelf and the books it holds.
Book: The title, authors, and reviews of the book.
Author: The name of the author and the books they have written.
Member: The name of the member and the books they have borrowed.
Loan: Information about borrowed books, including loan and return dates.
Implementation details
One-to-one relationship
Library - address: Each library has one address.
One-to-many relationship
Library - shelves: A library can have many shelves.
Shelf - books: A shelf can hold many books.
Many-to-many relationship
Book - authors: A book can have many authors, and an author can write many books.
Member - loans: A member can borrow many books, and many members can borrow a book through loans.


### One-to-one relationship
A one-to-one relationship exists when a single record in one table is associated with a single record in another table. For example, imagine a library system where each library has one unique address, and each address is linked to precisely one library. This relationship ensures that every library entry corresponds to one specific address.

Implementing a one-to-one relationship in a traditional relational database involves creating multiple tables and using foreign keys to establish references between them. This can get quite complex, especially when ensuring data integrity and uniqueness. Here’s an example of how you would set up a one-to-one relationship in SQL:

```rell
CREATE TABLE Address (
    id INT PRIMARY KEY,
    street VARCHAR(255),
    city VARCHAR(255),
    postalCode VARCHAR(10)
);

CREATE TABLE Library (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    address_id INT,
    FOREIGN KEY (address_id) REFERENCES Address(id)
);
```

In this example:

Address table: Contains the address details with a primary key ID to identify each address uniquely.
Library table: Contains library details and has a foreign key, address_id, which references the ID in the Address table. The UNIQUE constraint ensures that each address is linked to only one library.
Instead of manually dealing with foreign keys and constraints, Rell lets you directly reference entities.

```rell
entity library {
    library_name: text;     // Name of the library
    key address;    // One-to-one relationship with Address
}

entity address {
    street: text;         // Street address
    city: text;           // City name
    postal_code: text;    // Postal code
}
```

```sql
create table "c0.address"(
  "rowid" bigint not null,
  "street" text not null,
  "city" text not null,
  "postal_code" text not null,
  constraint "PK_c0.address"
    primary key ("rowid")
);

create table "c0.library"(
  "rowid" bigint not null,
  "name" text not null,
  "address" bigint not null,
  constraint "PK_c0.library"
    primary key ("rowid"),
  constraint "c0.library_address_FK"
    foreign key ("address")
    references "c0.address" ("rowid"),
  constraint "K_c0.library_0"
    unique ("address")
);
```


```rell
operation create_library(name: text, street: text, city: text, postal_code: text) {
    val library_address = create address(street, city, postal_code);
    create library(name, library_address);
}

query get_all_library() = library @* {} ( id = .rowid, .library_name, city = .address.city, street = .address.street, postal_code = .address.postal_code );

query get_library_by_name(library_name: text) = library @ { library_name };
```

Testing

```rell
@test module;

import library_management.{ create_library, get_all_library };

function test_create_library() {
    // Create a new library with an address
    rell.test.tx()
        .op(create_library("Biblioteca Apostolica Vaticana", "Città del Vaticano", "Vatican", "00120"))
        .op(create_library("New York Public Library", "476 5th Ave", "New York", "10018"))
        .run();

    val all_libraries = get_all_library();
    assert_equals(all_libraries.size(), 2);
}
```

### One-to-many relationships
One-to-many relationships represent a single record in one table linked to multiple records in another. For instance, in a library system, a single library can have numerous shelves, but each shelf belongs to only one library. This type of relationship is typically implemented using foreign keys in the child table (the table representing the “many” side of the relationship).

To better illustrate this concept, let’s explore how to define these relationships using SQL. We’ll consider the relationships between Library and Shelf, and Shelf and Book.

Here’s how it would look in SQL:

```sql
CREATE TABLE Library (
    id INT PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE Shelf (
    id INT PRIMARY KEY,
    number INT,
    library_id INT,
    FOREIGN KEY (library_id) REFERENCES Library(id)
);

CREATE TABLE Book (
    id INT PRIMARY KEY,
    title VARCHAR(255),
    shelf_id INT,
    FOREIGN KEY (shelf_id) REFERENCES Shelf(id)
);
```

In this example:

Library table: Contains information about libraries.
Shelf table: Includes a foreign key library_id to link each shelf to a specific library.
Book table: Has a foreign key shelf_id to associate each book with a specific shelf.
While this setup is effective, it requires careful management of foreign keys and constraints to ensure data integrity. Each step involves meticulous SQL syntax to maintain the relationships between tables. Defining the same entities and their relationships in Rell is more streamlined and expressive. Rell abstracts much of the complexity, making the code cleaner and easier to understand.

```rell
entity shelf {
    number: integer;    // Shelf number
    index library;      // Many-to-one relationship with Library
}

entity book {
    title: text;    // Title of the book
    index shelf;    // Many-to-one relationship with Shelf
}
```

```sql
create table "c0.shelf"(
  "rowid" bigint not null,
  "number" bigint not null,
  "library" bigint not null,
  constraint "PK_c0.shelf"
    primary key ("rowid"),
  constraint "c0.shelf_library_FK"
    foreign key ("library")
    references "c0.library" ("rowid")
);
create index "IDX_c0.shelf_0" on "c0.shelf"("library");

create table "c0.book"(
  "rowid" bigint not null,
  "title" text not null,
  "shelf" bigint not null,
  constraint "PK_c0.book"
    primary key ("rowid"),
  constraint "c0.book_shelf_FK"
    foreign key ("shelf")
    references "c0.shelf" ("rowid")
);
create index "IDX_c0.book_0" on "c0.book"("shelf");
```

```rell
operation create_shelf(number: integer, library_name: text) {
    val library = get_library_by_name(library_name);
    create shelf(number, library);
}

operation create_book(title: text, library_name: text, shelf_number: integer) {
    val library = get_library_by_name(library_name);
    val shelf = get_shelf_by_number(library, shelf_number);
    create book(title, shelf);
}

query get_shelf_by_library_name(id: rowid) = shelf @ { .rowid == id };

query get_book_by_title(title: text) = book @ { .title == title };

query get_shelf_by_number(library, number: integer) = shelf @ { .library == library, .number == number };

query get_all_shelves() = shelf @* { } ( id = .rowid, library_name = .library.library_name, .number );

query get_all_books() = book @* { } ( id = .rowid, .title );

query get_libraries_with_shelves() {
    return (l: library, @outer s: shelf @* { s.library == l }) @* {} ( @group l.library_name, @list shelves = s?.number );
}
```

Testing

```rell
@test module;

import library_management.{ create_library, get_all_library, create_shelf, get_all_shelves, create_book, get_all_books };

function test_create_library() {
    rell.test.tx()
        .op(create_library("Biblioteca Apostolica Vaticana", "Città del Vaticano", "Vatican", "00120"))
        .op(create_library("New York Public Library", "476 5th Ave", "New York", "10018"))
        .run();

    val all_libraries = get_all_library();
    assert_equals(all_libraries.size(), 2);

    val library_name = all_libraries[1].library_name;
    for (n in range(1, 4)) {
        rell.test.tx()
            .op(create_shelf(n, library_name))
            .run();
    }

    val all_shelves = get_all_shelves();
    assert_equals(all_shelves.size(), 3);

    rell.test.tx()
        .op(create_book("To Kill a Mockingbird", "New York Public Library", 1))
        .op(create_book("1984", "New York Public Library", 2))
        .op(create_book("Pride and Prejudice", "New York Public Library", 3))
        .run();

    val all_books = get_all_books();
    assert_equals(all_books.size(), 3);
}
```

### Many-to-many relationships

A many-to-many relationship exists when multiple records in one table are associated with multiple records in another table. For example, a book can have multiple authors, and an author can write multiple books. This relationship is typically implemented using a junction table (or join table) that contains foreign keys referencing the primary keys of the two tables involved in the relationship.

Now, let’s define entities that represent many-to-many relationships. These relationships ensure that each book can have multiple authors, and each author can write multiple books. Similarly, each member can borrow multiple books, and multiple members can borrow each book. This structure maintains the integrity and organization of the data within our library management application.

In a traditional relational database, many-to-many relationships are established using a join table. Here’s how it would look in SQL:

```sql
CREATE TABLE Author (
    id INT PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE Book (
    id INT PRIMARY KEY,
    title VARCHAR(255)
);

CREATE TABLE Book_Author (
    book_id INT,
    author_id INT,
    PRIMARY KEY (book_id, author_id),
    FOREIGN KEY (book_id) REFERENCES Book(id),
    FOREIGN KEY (author_id) REFERENCES Author(id)
);

CREATE TABLE Member (
    id INT PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE Loan (
    member_id INT,
    book_id INT,
    PRIMARY KEY (member_id, book_id),
    FOREIGN KEY (member_id) REFERENCES Member(id),
    FOREIGN KEY (book_id) REFERENCES Book(id)
);
```

```rell
entity author {
    key author_name: text;    // Name of the author
}

entity book_author {
    key book, author;  // Composite key to establish many-to-many relationship
}

entity member {
    key member_name: text;    // Name of the member
}

entity loan {
    key member, book;          // Composite key to establish many-to-many relationship
}
```

```sql
create table "c0.author"(
  "rowid" bigint not null,
  "name" text not null,
  constraint "PK_c0.author"
    primary key ("rowid"),
  constraint "K_c0.author_0"
    unique ("name")
);

create table "c0.book_author"(
  "rowid" bigint not null,
  "book" bigint not null,
  "author" bigint not null,
  constraint "PK_c0.book_author"
    primary key ("rowid"),
  constraint "c0.book_author_book_FK"
    foreign key ("book")
    references "c0.book" ("rowid"),
  constraint "c0.book_author_author_FK"
    foreign key ("author")
    references "c0.author" ("rowid"),
  constraint "K_c0.book_author_0"
    unique (
      "book",
      "author"
    )
);

create table "c0.member"(
  "rowid" bigint not null,
  "name" text not null,
  constraint "PK_c0.member"
    primary key ("rowid"),
  constraint "K_c0.member_0"
    unique ("name")
);

create table "c0.loan"(
  "rowid" bigint not null,
  "member" bigint not null,
  "book" bigint not null,
  constraint "PK_c0.loan"
    primary key ("rowid"),
  constraint "c0.loan_member_FK"
    foreign key ("member")
    references "c0.member" ("rowid"),
  constraint "c0.loan_book_FK"
    foreign key ("book")
    references "c0.book" ("rowid"),
  constraint "K_c0.loan_0"
    unique (
      "member",
      "book"
    )
);
```

```rell
operation create_author(authors_name: text) {
    create author(authors_name);
}

operation create_book_author(title: text, authors_name: text) {
    val new_book = get_book_by_title(title);
    val new_author = get_author_by_name(authors_name);
    create book_author(new_book, new_author);
}

operation create_member(name: text) {
    create member(name);
}

operation create_loan(member_name: text, book_title: text) {
    val member = get_member_by_name(member_name);
    val book = get_book_by_title(book_title);
    create loan (member, book);
}


query get_all_authors() = author @* { } ( .author_name );

query get_all_members() = member @* { } ( id = .rowid, .member_name );

query get_all_loans() = loan @* { } ( id = .rowid );

query get_member_by_name(name: text) = member @ { .member_name == name };

query get_author_by_name(author_name: text) = author @ { .author_name == author_name };

query get_all_books_with_authors() = book_author @* {} ( book_id = .book.rowid, author_id = .author.rowid);
```

Testing

```rell
@test module;

import library_management.{ create_library, get_all_library, create_shelf, get_all_shelves, create_book_author, get_all_books, get_all_authors, create_book, create_author, get_all_books_with_authors, create_member, get_all_members, create_loan, get_all_loans };

function test_create_library() {
    rell.test.tx()
        .op(create_library("Biblioteca Apostolica Vaticana", "Città del Vaticano", "Vatican", "00120"))
        .op(create_library("New York Public Library", "476 5th Ave", "New York", "10018"))
        .run();

    val all_libraries = get_all_library();
    assert_equals(all_libraries.size(), 2);

    val library_name = all_libraries[1].library_name;
    for (n in range(1, 4)) {
        rell.test.tx()
            .op(create_shelf(n, library_name))
            .run();
    }

    val all_shelves = get_all_shelves();
    assert_equals(all_shelves.size(), 3);

    rell.test.tx()
        .op(create_book("To Kill a Mockingbird", "New York Public Library", 1))
        .op(create_book("1984", "New York Public Library", 2))
        .op(create_book("Pride and Prejudice", "New York Public Library", 3))
        .op(create_book("The Great Gatsby", "New York Public Library", 3))
        .run();

    val all_books = get_all_books();
    assert_equals(all_books.size(), 4);

    rell.test.tx()
        .op(create_author("Harper Lee"))
        .op(create_author("George Orwell"))
        .op(create_author("Jane Austen"))
        .op(create_author("F. Scott Fitzgerald"))
        .run();

    val all_authors = get_all_authors();
    assert_equals(all_authors.size(), 4);

    rell.test.tx()
        .op(create_book_author("To Kill a Mockingbird", "Harper Lee"))
        .op(create_book_author("1984", "George Orwell"))
        .op(create_book_author("Pride and Prejudice", "Jane Austen"))
        .op(create_book_author("The Great Gatsby", "F. Scott Fitzgerald"))
        .run();

    val all_books_with_authors = get_all_books_with_authors();
    assert_equals(all_books_with_authors.size(), 4);

    rell.test.tx()
        .op(create_member("Erik"))
        .run();

    rell.test.tx()
        .op(create_member("Mark"))
        .run();

    val all_members = get_all_members();
    assert_equals(all_members.size(), 2);

    rell.test.tx()
        .op(create_loan("Erik", "To Kill a Mockingbird"))
        .op(create_loan("Erik", "1984"))
        .op(create_loan("Erik", "Pride and Prejudice"))
        .run();

    rell.test.tx()
        .op(create_loan("Mark", "The Great Gatsby"))
        .run();

    val all_loans = get_all_loans();
    assert_equals(all_loans.size(), 4);
}
```

### Join
Types of joins covered:

Inner Join: Returns records that have matching values in both entities.
Outer Join: Returns all records from both entities, including those that do not have a match in the other entity. If there is no match, the result is NULL on the side with no corresponding record.

```rell
// Query to retrieve books along with their authors using an inner join
query get_books_with_authors() {
    return (book, book_author, author) @* {
        book == book_author.book,
        author == book_author.author
    } (book.title, author.author_name);
}

query get_libraries_with_shelves() {
    return (l: library, @outer s: shelf @* { s.library == l }) @* {} ( @group l.library_name, @list shelves = s?.number );
}

// Query to retrieve books, their authors, and the libraries they are located in
query get_books_authors_libraries() {
    return (book, book_author, author, shelf, library) @* {
        book == book_author.book,
        author == book_author.author,
        book.shelf == shelf,
        shelf.library == library
    } (book.title, author.author_name, library.library_name);
}

// Query to retrieve members along with the books they have on loan using an inner join
query get_members_with_loans() {
    return (member, loan, book) @* {
        member == loan.member,
        book == loan.book
    } (member.member_name, book.title);
}

// Query to retrieve books on loan along with the borrowing members and the authors of the books
query get_books_on_loan() {
    return (book, loan, member, book_author, author) @* {
        book == loan.book,
        member == loan.member,
        book == book_author.book,
        author == book_author.author
    } (book.title, member.member_name, author.author_name);
}
```

## Example: Store Prompt Histories

```rell
module;

import lib.ft4.utils.{latest_time};
import production.{owner};

// ================================
// Entity
// ================================
entity prompt_history {
    prompt_id: integer;
    key prompt_id;
    UID: integer; // scout id
    messages: json; // input json
    mutable result: json; // output json
    seed: integer;
    model: text;
    provider: text;
    created_at: integer = latest_time();
}

// ================================
// Operation
// ================================
struct Prompt {
    UID: integer;
    messages: json;
    result: json;
    seed: integer;
    model: text;
    provider: text;
}

struct prompt_dto {
    prompt_id: integer;
    UID: integer;
    messages: json;
    result: json;
    seed: integer;
    model: text;
    provider: text;
    created_at: integer;
}

operation add_prompt_history(
    UID: integer,
    messages: json,
    result: json,
    seed: integer,
    model: text,
    provider: text,
) {
    is_owner();
    val prompt_id = owner.prompt_id + 1;
    owner.prompt_id = prompt_id;
    create prompt_history (
        prompt_id,
        UID,
        messages,
        result,
        seed,
        model,
        provider,
    );
}

operation update_prompt_history(
    prompt_id: integer,
    result: json,
) {
    is_owner();
    require(
        prompt_history @? {
            .prompt_id == prompt_id
        } != null,
        "Entity with prompt_id %s not found"
            .format(
                prompt_id
            )
    );
    update prompt_history @ { .prompt_id == prompt_id } (
        result = result,
    );
}

operation delete_prompt_history(
    prompt_id: integer,
) {
    is_owner();
    require(
        prompt_history @? {
            .prompt_id == prompt_id
        } != null,
        "Entity with prompt_id %s not found"
            .format(
                prompt_id
            )
    );
    delete prompt_history @ { .prompt_id == prompt_id };
}

operation validate_user_with_no_operation() {
    is_owner();
}

operation change_owner(new_address: pubkey) {
    is_owner();
    require(op_context.is_signer(new_address), "New owner must sign the transaction");
    owner.address = new_address;
}

operation batch_delete_prompt_histories(
    start_time: integer,
    end_time: integer,
) {
    is_owner();
    val prompt_histories = get_prompt_histories(start_time, end_time, 0, 100);
    for (p in prompt_histories.prompts) {
        delete prompt_history @ { .prompt_id == p.prompt_id };
    }
}

// ================================
// Query
// ================================
query latest_prompt_id() {
    val latest_prompt = prompt_history @? { } (
        @sort_desc @omit .created_at,
        prompt_dto (
            prompt_id = .prompt_id,
            UID = .UID,
            messages = .messages,
            result = .result,
            seed = .seed,
            model = .model,
            provider = .provider,
            created_at = .created_at,
        )
    ) limit 1;
    var prompt_id = 0;
    if (latest_prompt != null) {
        prompt_id = latest_prompt.prompt_id;
    }
    return prompt_id;
}

query get_prompt_history(
    prompt_id: integer,
) {
    return prompt_history @ {
        .prompt_id == prompt_id
    } (
        prompt_dto (
            prompt_id = .prompt_id,
            UID = .UID,
            messages = .messages,
            result = .result,
            seed = .seed,
            model = .model,
            provider = .provider,
            created_at = .created_at,
        )
    );
}

query get_prompt_history_count() {
    return prompt_history @ { } (
        @sum 1
    );
}

query get_prompt_histories(
    start_time: integer,
    end_time: integer,
    pointer: integer,
    n_prompts: integer,
): (pointer: integer,prompts: list<prompt_dto>) {
    val prompts = prompt_history @* {
        .created_at > start_time,
        .created_at < end_time,
    } (
        @sort_desc @omit .created_at,
        prompt_dto (
            prompt_id = .prompt_id,
            UID = .UID,
            messages = .messages,
            result = .result,
            seed = .seed,
            model = .model,
            provider = .provider,
            created_at = .created_at,
        )
    ) offset pointer limit n_prompts;
    return (
        pointer = pointer + prompts.size(),
        prompts = prompts
    );
}

query get_prompt_histories_by_uid(
    UID: integer,
    n_prompts: integer,
    pointer: integer,
): (pointer: integer, prompts: list<prompt_dto>) {
    val prompts = prompt_history @* {
        .UID == UID
    } (
        @sort_desc @omit .created_at,
        prompt_dto (
            prompt_id = .prompt_id,
            UID = .UID,
            messages = .messages,
            result = .result,
            seed = .seed,
            model = .model,
            provider = .provider,
            created_at = .created_at,
        )
    ) offset pointer limit n_prompts;
    return (
        pointer = pointer + prompts.size(),
        prompts = prompts
    );
}

query get_owner() {
    return owner.address;
}

// ================================
// Util
// ================================
function is_owner() {
    require(op_context.is_signer(owner.address), "Only the owner can call this operation");
}
```

## Example: Store OPENAI logs on Chromia via RELL

```rell
module;

object owner {
    mutable address: pubkey = x"03EE16E285050088557DF422D0CF55736C4C355999E64D2D3547CFB581CDC3136B";
}

function is_owner() {
    require(op_context.is_signer(owner.address), "Only the owner can call this operation");
}

entity allowlist {
    address: byte_array;
    key address;
}

query get_top_allowlist() {
    return allowlist @* {} (
        @sort_desc @omit .address,
        address= .address
    ) offset 0 limit 10;
}

query check_allowlist(address: byte_array): allowlist? {
    return allowlist @? { .address == address };
}

operation add_allowlist(address: byte_array) {
    is_owner();
    create allowlist(address);
}

operation remove_allowlist(address: byte_array) {
    is_owner();
    delete allowlist @ { .address == address };
}

function is_allowlist(): byte_array {
    val signer = op_context.get_signers()[0];
    require(allowlist @? { .address == signer }, "Only allowlisted users can call this operation");
    return signer;
}

entity open_ai_logs {
    address: byte_array;

    uuid: text;
    key uuid;
    chat_id: text;

    // REQUEST
    base_url: text;
    request_model: text;
    request_messages: json;
    user_question: text;
    request_raw: json;

    // RESPONSE
    response_object: text;
    response_created: integer;
    response_model: text;
    response_system_fingerprint: text;
    response_provider: text;
    response_usage_prompt_tokens: integer;
    response_usage_completion_tokens: integer;
    response_usage_total_tokens: integer;
    assistant_reply: text;
    finish_reason: text;
    response_raw: json;
    created_at: integer = op_context.last_block_time;
}

struct open_ai_logs_dto {
    address: byte_array;
    uuid: text;
    chat_id: text;
    base_url: text;
    request_model: text;
    request_messages: json;
    user_question: text;
    request_raw: json;
    response_object: text;
    response_created: integer;
    response_model: text;
    response_system_fingerprint: text;
    response_provider: text;
    response_usage_prompt_tokens: integer;
    response_usage_completion_tokens: integer;
    response_usage_total_tokens: integer;
    assistant_reply: text;
    finish_reason: text;
    response_raw: json;
    created_at: integer;
}

operation add_log(
    chat_id: text,
    base_url: text,
    request_model: text,
    request_messages: json,
    user_question: text,
    request_raw: json,
    response_object: text,
    response_created: integer,
    response_model: text,
    response_system_fingerprint: text,
    response_provider: text,
    response_usage_prompt_tokens: integer,
    response_usage_completion_tokens: integer,
    response_usage_total_tokens: integer,
    assistant_reply: text,
    finish_reason: text,
    response_raw: json
) {
    val addr = is_allowlist();
    val uuid = request_model + "-" + chat_id;
    create open_ai_logs (
        address = addr,
        uuid,
        chat_id,
        base_url,
        request_model,
        request_messages,
        user_question,
        request_raw,
        response_object,
        response_created,
        response_model,
        response_system_fingerprint,
        response_provider,
        response_usage_prompt_tokens,
        response_usage_completion_tokens,
        response_usage_total_tokens,
        assistant_reply,
        finish_reason,
        response_raw
    );
}

query get_log(uuid: text): open_ai_logs_dto {
    return open_ai_logs @ {
        .uuid == uuid
    } (
        open_ai_logs_dto (
            .address,
            .uuid,
            .chat_id,
            .base_url,
            .request_model,
            .request_messages,
            .user_question,
            .request_raw,
            .response_object,
            .response_created,
            .response_model,
            .response_system_fingerprint,
            .response_provider,
            .response_usage_prompt_tokens,
            .response_usage_completion_tokens,
            .response_usage_total_tokens,
            .assistant_reply,
            .finish_reason,
            .response_raw,
            .created_at
        )
    );
}

query get_logs_count() {
    return open_ai_logs @ { } (
        @sum 1
    );
}

query get_logs(
    address: byte_array,
    start_time: integer,
    end_time: integer,
    pointer: integer,
    n_prompts: integer,
): (pointer: integer, logs: list<open_ai_logs_dto>) {
    val logs = open_ai_logs @* {
        .address == address,
        .created_at > start_time,
        .created_at < end_time,
    } (
        @sort_desc @omit .created_at,
        open_ai_logs_dto (
            address = .address,
            uuid = .uuid,
            chat_id = .chat_id,
            base_url = .base_url,
            request_model = .request_model,
            request_messages = .request_messages,
            user_question = .user_question,
            request_raw = .request_raw,
            response_object = .response_object,
            response_created = .response_created,
            response_model = .response_model,
            response_system_fingerprint = .response_system_fingerprint,
            response_provider = .response_provider,
            response_usage_prompt_tokens = .response_usage_prompt_tokens,
            response_usage_completion_tokens = .response_usage_completion_tokens,
            response_usage_total_tokens = .response_usage_total_tokens,
            assistant_reply = .assistant_reply,
            finish_reason = .finish_reason,
            response_raw = .response_raw,
            created_at = .created_at
        )
    ) offset pointer limit n_prompts;
    return (
        pointer = pointer + logs.size(),
        logs = logs
    );
}
```