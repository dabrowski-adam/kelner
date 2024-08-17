# Kelner

**Table of Contents**

- [About](#about)
- [Implementation](#implementation)
- [Usage](#usage)
- [Meta](#meta)


## About

Kelner is a helper for generating DML query params from domain objects. 

## Implementation

<details>
<summary>Source code</summary>

```scala
type TupleConsistsOf[A <: Tuple, B] = A match
    case B *: tail  => TupleConsistsOf[tail, B]
    case EmptyTuple => DummyImplicit
    case _          => Nothing

type Of[A] = [X <: Tuple] =>> TupleConsistsOf[X, A]
```

```scala
type Column[VALUE] = (String & Singleton, VALUE)

type ColumnNames[T <: NonEmptyTuple] <: Tuple = T match
    case (name, ?) *: tail => name *: ColumnNames[tail]
    case _                 => EmptyTuple

trait Mapping[-DOMAIN, ROW <: (String & Singleton, NonEmptyTuple)]:
    def encode(a: DOMAIN): Tuple.Elem[ROW, 1]

trait Table[NAME <: String & Singleton, COLUMNS <: NonEmptyTuple : Of[Column[?]]]:
    type Columns = COLUMNS
    type Row     = (NAME, COLUMNS)
    
    given CanEqual[Tuple.Union[Columns], Tuple.Union[Columns]] = CanEqual.derived
    
    def primaryKey: List[Tuple.Union[ColumnNames[Columns]]]
    
    def params[A](data: A)(using e: Mapping[A, Row]): List[Tuple.Union[Columns]] =
        e.encode(data).toList
    
    def diff[A](
      before:            A,
      after:             A,
      includePrimaryKey: Boolean = false,
    )(
      using Mapping[A, Row],
    ): List[Tuple.Union[Columns]] =
        val columnsBefore = params(before)
        val columnsAfter  = params(after)
        
        columnsBefore.zip(columnsAfter).collect:
            case ((name, _), y) if includePrimaryKey && primaryKey.contains(name) => y 
            case (x,         y) if x != y                                         => y
```

</details>

## Usage

Define a table and a way of transforming your domain objects into rows:

```scala
object Users extends Table["users", (("id", Int), ("name", String))]:
    override def primaryKey = List("id")

case class User(id: Int, name: String)

given Mapping[User, Users.Row] = (user: User) => (
    "id"   -> user.id,
    "name" -> user.name,
)
```

Use `Table` methods to prepare parameters for your queries:

```scala
val user = User(id = 0, name = "Adam Dąbrowski")
// user: User = User(id = 0, name = "Adam Dąbrowski")

val insertParams = Users.params(user)
// insertParams: List[Tuple2["id", Int] | Tuple2["name", String]] = List(
//   ("id", 0),
//   ("name", "Adam Dąbrowski")
// )

val updatedUser = user.copy(name = "A. D.")
// updatedUser: User = User(id = 0, name = "A. D.")

val updateParams = Users.diff(user, updatedUser)
// updateParams: List[Tuple2["id", Int] | Tuple2["name", String]] = List(
//   ("name", "A. D.")
// )

val updateParamsWithPrimaryKey = Users.diff(user, updatedUser, includePrimaryKey = true)
// updateParamsWithPrimaryKey: List[Tuple2["id", Int] | Tuple2["name", String]] = List(
//   ("id", 0),
//   ("name", "A. D.")
// )
```

The compiler will check if `primaryKey` references defined columns.

```scala ignore
object Reactions extends Table["reactions", (("post_id", Int), ("user_id", Int))]:
    override def primaryKey = List("id", "user_id")
// error:
// Found:    ("id" : String)
// Required: ("post_id" : String) | ("user_id" : String)
//     override def primaryKey = List("id", "user_id")
//                                    ^^^^
```

It will also ensure that you provide a valid transformation from domain objects to rows.

```scala ignore
object Reactions extends Table["reactions", (("post_id", Int), ("user_id", Int))]:
    override def primaryKey = List("post_id", "user_id")

case class Reaction(id: Int, userId: Int)

given Mapping[Reaction, Reactions.Row] = (reaction: Reaction) => (
    "id"      -> reaction.id,
    "user_id" -> reaction.userId,
)
// error:
// Found:    (String, Int)
// Required: (("post_id" : String), Int)
//     "id"      -> reaction.id,
//     ^^^^^^^^^^^^^^^^^^^^^^^^
```

If you have two tables with the same columns, you can't accidentally swap them.

```scala ignore
object Items extends Table["items", (("id", Int), ("name", String))]:
    override def primaryKey = List("id")

case class Item(id: Int, name: String)

given Mapping[Item, Items.Row] = (item: Item) => (
    "id"   -> item.id,
    "name" -> item.name,
)

Items.params(user)
// error:
// No given instance of type repl.MdocSession.MdocApp.Mapping[repl.MdocSession.MdocApp.User,
//   repl.MdocSession.MdocApp.Items.Row] was found for parameter e of method params in trait Table
// Items.params(user)
//                  ^
```


## Meta

This readme was generated from `readme.scala.md` using `scala-cli --power readme.scala.md`.

```scala raw
//> using jvm 22
//> using scala 3.5.0-RC7
//> using mainClass Main

//> using options -deprecation -feature -language:strictEquality
//> using options -Xmax-inlines:64 -Xkind-projector:underscores
//> using options -Yexplicit-nulls -Ysafe-init-global
//> using options -Wsafe-init -Wnonunit-statement -Wshadow:all

//> using dep org.scalameta::mdoc:2.5.4
```

```scala raw
object Main:
    def main(args: Array[String]): Unit =
        val classpath = System.getProperty("java.class.path")
        val mdocArgs  = List(
            "--classpath", classpath,
            "--in", "readme.scala.md",
            "--out", "readme.md",
        )
        val settings  = mdoc.MainSettings().withArgs(args.toList ++ mdocArgs)
        val exitCode  = mdoc.Main.process(settings)
        
        if (exitCode != 0) sys.exit(exitCode)
```
