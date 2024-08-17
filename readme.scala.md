#!/usr/bin/env -S scala-cli --power shebang
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

```scala mdoc
type TupleConsistsOf[A <: Tuple, B] = A match
    case B *: tail  => TupleConsistsOf[tail, B]
    case EmptyTuple => DummyImplicit
    case _          => Nothing

type Of[A] = [X <: Tuple] =>> TupleConsistsOf[X, A]

type Column[VALUE] = (String & Singleton, VALUE)

type ColumnNames[T <: NonEmptyTuple] <: Tuple = T match
    case (name, ?) *: tail => name *: ColumnNames[tail]
    case _                 => EmptyTuple

trait Mapping[-DOMAIN, ROW <: (String & Singleton, NonEmptyTuple)]:
    def encode(a: DOMAIN): Tuple.Elem[ROW, 1]

trait Table[NAME <: String & Singleton : ValueOf, COLUMNS <: NonEmptyTuple : Of[Column[?]]]:
    type Columns = COLUMNS
    type Row     = (NAME, COLUMNS)
    
    given CanEqual[Tuple.Union[Columns], Tuple.Union[Columns]] = CanEqual.derived
    
    def primaryKey: List[Tuple.Union[ColumnNames[Columns]]]

    def name: NAME = valueOf[NAME]
    
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

```scala mdoc
object Users extends Table["users", (("id", Int), ("name", String))]:
    override def primaryKey = List("id")

case class User(id: Int, name: String)

given Mapping[User, Users.Row] = (user: User) => (
    "id"   -> user.id,
    "name" -> user.name,
)
```

Use `Table`'s `params` and `diff` methods to prepare parameters for your queries:

```scala mdoc
val user = User(id = 0, name = "Adam Dąbrowski")

val insertParams = Users.params(user)

val updatedUser = user.copy(name = "A. D.")

val updateParams = Users.diff(user, updatedUser)

val updateParamsWithPrimaryKey = Users.diff(user, updatedUser, includePrimaryKey = true)
```

The compiler will check if `primaryKey` references defined columns.

```scala mdoc:fail ignore
object Reactions extends Table["reactions", (("post_id", Int), ("user_id", Int))]:
    override def primaryKey = List("id", "user_id")
```

It will also ensure that you provide a valid transformation from domain objects to rows.

```scala mdoc:fail ignore
object Reactions extends Table["reactions", (("post_id", Int), ("user_id", Int))]:
    override def primaryKey = List("post_id", "user_id")

case class Reaction(id: Int, userId: Int)

given Mapping[Reaction, Reactions.Row] = (reaction: Reaction) => (
    "id"      -> reaction.id,
    "user_id" -> reaction.userId,
)
```

If you have two tables with the same columns, you can't accidentally swap them.

```scala mdoc:fail ignore
object Items extends Table["items", (("id", Int), ("name", String))]:
    override def primaryKey = List("id")

case class Item(id: Int, name: String)

given Mapping[Item, Items.Row] = (item: Item) => (
    "id"   -> item.id,
    "name" -> item.name,
)

Items.params(user)
```

# Use cases

## Generate partial updates

```scala mdoc
 // Implementation making use of com.datastax.driver.core.BoundStatement is left as an exercise to the reader.
type CassandraQuery = String

trait Cassandra:
    this: Table[?, ?] =>
        
        def update[A](before: A, after: A)(using Mapping[A, this.Row]): CassandraQuery =
            val params    = this.diff(before, after, includePrimaryKey = true)
            val columns   = params.map:
                case (name, _value) => name
            val values    = params.map:
                case (_name, value: String) => s"'$value'"
                case (_name, value: Int)    => value
            
            s"INSERT (${columns.mkString(", ")}) INTO ${this.name} VALUES (${values.mkString(", ")})"
```

```scala mdoc
object Posts extends Table[
    "posts",
    (
        ("id",         Int),
        ("content",    String),
        ("created_at", String),
    )
] with Cassandra:
    override def primaryKey = List("id")

case class Post(id: Int, content: String, createdAt: String)

given Mapping[Post, Posts.Row] = (post: Post) => (
  "id"         -> post.id,
  "content"    -> post.content,
  "created_at" -> post.createdAt,
)

val post = Post(id = 0, content = "Lorem ipsum...", createdAt = "timestamp")

val updatedPost = post.copy(content = "Quidquid latine dictum sit, altum videtur.")

val cassandraUpdateQuery = Posts.update(post, updatedPost)
```

## Meta

This readme was generated from [readme.scala.md](readme.scala.md?plain=1). 

First, make it executable (yes!) with `chmod +x readme.scala.md`, then just run it: `./readme.scala.md`. 😀

```scala mdoc:invisible raw
//> using jvm 22
//> using scala 3.5.0-RC7
//> using mainClass Main

//> using options -deprecation -feature -language:strictEquality
//> using options -Xmax-inlines:64 -Xkind-projector:underscores
//> using options -Yexplicit-nulls -Ysafe-init-global
//> using options -Wsafe-init -Wnonunit-statement -Wshadow:all

//> using dep org.scalameta::mdoc:2.5.4
```

```scala mdoc:invisible raw
import scala.util.{Try, Success, Failure, Using}
import scala.io.Source
import java.io.PrintWriter

object Main:
   
    def main(args: Array[String]): Unit =
        val classpath  = System.getProperty("java.class.path")
        val inputFile  = "readme.scala.md"
        val outputFile = inputFile.replace(".scala.md", ".md")
        
        val mdocArgs  = List(
            "--classpath", classpath,
            "--in", inputFile,
            "--out", outputFile,
        )
        val settings  = mdoc.MainSettings().withArgs(args.toList ++ mdocArgs)
        
        val program =
            for
                _ <- mdoc.Main.process(settings) match
                         case 0        => Success(())
                         case exitCode => Failure(new RuntimeException(s"mdoc failed with exit code $exitCode"))
                _ <- trimShebang(outputFile)
            yield ()
        
        val exitCode = program.fold(_failure => 1, _success => 0)
        sys.exit(exitCode)
    
    
    def trimShebang(filePath: String): Try[Unit] =
        Using.Manager: use =>
            val lines  = use(Source.fromFile(filePath)).getLines().toList
            val writer = use(new PrintWriter(filePath))
            
            lines.dropWhile(_.startsWith("#!")).foreach(writer.println)
```
