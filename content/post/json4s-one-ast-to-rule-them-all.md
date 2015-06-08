---
title: "Json4s: One AST to rule them all."
date: 2012-10-25T08:48:23Z
comments: true
categories: scala json json4s
---

It seems to me that every webframework in Scala, of which we have plenty, also insists on writing their own JSON library.
But various tools rely on the AST from lift-json.  This is both good and bad, there were a number of gaps in the lift-json version.
The most notable being that it does not support any other type than Double for representing decimal numbers.
A second problem is that because of the large number of dependencies in the lift project it typically is a bottleneck for upgrading to scala-2.10.
And thirdly it just seems an odd place for a nice library like that to live.
I'm hopeful that more libraries like Play, Spray etc will contribute to this project so that instead of a fragmented json landscape we get a homogeneous one. All of them have a json ast with the same types defined in it.

So I've set out to set lift-json free from the lift project and tried to add some improvements along the way.
The first improvement I've made is that you now have the choice between using `BigDecimal` or `Double` for representing decimal numbers, so you can use the library also to represent invoices etc.
The second change I made is to add several backends for parsing. The original lift-json parser is still available in the native package but you can now also use jackson as a parsing backend to the json library.
It's really easy to add more backends so smarterjson, spray-json etc are all in the cards.

I took a look at what play2 has in their json support and it looks like their main thing is a type class based reader/writer story instead of the formats based one from lift-json, so for good measure I also added that system to this library. In general I like typeclasses for this type of stuff but in this case I actually think that lift-json has a nicer approach by assembling all the context into a single formats object instead of requiring many type classes to be in scope at any given time when you want to parse or write json.

There are a few more convenience methods added on the `JValue` type that allow you to `camelize` or `underscore` keys, remove the nulls etc.
I spoke with [Joni Freeman](https://github.com/jonifreeman) about the general idea of this library and he showed me what he did on a branch for lift-json 3.0 so I incorporated his work into json4s too. it basically means that a key/value pair used to represent a `JObject` is no longer a valid json AST node and there are some extra methods to work with those fields. All of this is explained in the [README](https://github.com/json4s/json4s/blob/master/README.md) of the json4s project.

I've also added support for json4s to the [dispatch reboot](https://github.com/dispatch/reboot) project so you can use it there just like you can with lift-json. Furthermore [Rose Toomey](https://github.com/rktoomey) let me know that [salat](https://github.com/novus/salat) is now using json4s as json library instead of lift-json.

Some of the improvements I still want to make is for scala 2.10 I want to use the `Mirror` api to be able to reflect over more stuff than just case classes. For some of the use cases we have at where I work it makes sense to be able to use a few annotations (yes annotations unfortunately) to have certain keys be ignored and so on.  I'll probably steal some of that from the Salat project so that there is still some degree of consistency between our libraries.

I also want to figure out how we can possibly make the AST based approach useful with huge data structures, as it's not inconceivable to want to send 100MB or 10GB json docs over the wire. At that moment a lazy approach actually makes a lot of sense, I'm open to suggestions on how this could be achieved efficiently without breaking the AST model.

So if you're using json in scala consider using or contributing to the [json4s project](https://github.com/json4s/json4s).
