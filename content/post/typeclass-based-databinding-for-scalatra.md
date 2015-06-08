---
title: "Typeclass based databinding for scalatra"
date: 2012-09-08T09:14:43Z
comments: true
categories:
- Scala
- sbt
- Scalatra
tags:
- scala
- sbt
- scalatra
---

## Typeclass based databinding for scalatra

One of the big new features for scalatra in the next release will be databinding. We chose to use a command approach which includes validation.
It is built on top of [scalaz 6.0.4](https://github.com/scalaz/scalaz) at the moment, as soon as scalaz7 is released we'll probably migrate to that.

One of the tenets of scalatra is that it's a mutable interface, the request and response of servlets are both mutable classes and we depend on those. I also wanted for the commands to be defined with vals that you can address later.
The core functionality of the bindings is actually immutable but to the consumer of the library it pretends to be a mutable interface. Perhaps those properties will make people frown, perhaps not.

### What does it do?

It provides a way to represent input forms with validation attached to the fields, maybe a code sample will be a better explanation.

```scala
import org.json4s._
import org.json4s.jackson._
import org.scalatra.databinding._

class LoginForm extends JacksonCommand {
  val login: Field[String] = asString("login").notBlank
  val password: Field[String] = asString("password").notBlank
}
```

This interface relies on a bunch of implicit conversions to be in scope. To be able to present this mutable interface it is needed to use the `Field[T]` label after the val declaration otherwise things won't work as you'd expect them to work.

Then in the scalatra application you can do

```scala
import scalaz._
import Scalaz._

class LoginApp extends ScalatraServlet with JacksonJsonSupport with JacksonParsing {
  post("/login") {
    val cmd = command[LoginCommand]
    if (cmd.isValid) {
      users.login(~cmd.login.value, ~cmd.password.value)
    } else {
      halt(401, "invalid username/password")
    }
  }
}
```

This will work with data provided as form params in a http post, as json or as xml. Of course the example above is a simple one and there are many other propeties you can interrogate. At this moment the databinding is a bit light on documentation as in there is none. If you'd like to take it for a spin feel free to ping us in the #scalatra irc room.


### What's next?

At some point in the future I'd like turn a command into a monoid so that we have a map method on it, but for now I need to move onto another part of scalatra.
