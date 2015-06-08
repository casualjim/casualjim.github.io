---
title: "Scalatra 2.2 released"
date: 2013-02-13T11:30:15Z
comments: true
categories: scala scalatra
---

Last week we released Scalatra 2.2. This is our biggest release so far and introduces a bunch of exciting features like commands for handling input, atmosphere for websockets and comet. It has a much deeper swagger integration now and that API has been completely upgraded.
In short this scalatra version fixes most of the big problems we were aware off. Probably one of the nastiest of those problems was the fact that we were using thread-locals to store the request and response, when you then use a future or something the request is no longer available. Let's walk through some of these changes.

#### Working around thread-locals.

In a previous version we had migrated all our internal state to either the servlet context attributes or the request attributes, depending on their scope. In this release we made everything that accesses the request or response take them as implicit parameters. For people overriding our methods this is a breaking change, but easily fixed by adding the parameters to your method override. We also added an AsyncResult construct whose only purpose is to help you not close over thread-locals.

So what is exactly the problem?

```scala

def skip = params.getAsOrElse("skip", 0)

get("/things/:id") {
  Future {
    params("id")  // throws NPE because request is not available when the future executes
  }
}

post("/things") {
  myThingsActor ? Post(parsedBody.extract[Things]) map { things =>
    if (things.isEmpty) status = 404 // throws NPE because response is not available when the future executes
    ()
  }
}

// assuming scentry is mixed in and user is something stored on the request or in cookies or something
get("/stuff/:id") {
  val stuff: Future[Stuff] = getStuff(params("id"))
  // everything is still fine
  stuff map { allTheThings =>
    getTrinketsForUser(allTheThings, user, skip)  // throws NPE because request is not available when the future executes
  }
}
```

And since this is something we absolutely had to fix, we had to introduce some breaking changes but they really were for the better.
Currently there are 2 ways to get around it: bring request/response into your action in implicit vals or use the AsyncResult trait to do this for you.

Let's rewrite the broken examples in terms of the first work around:

```scala

def skip(implicit request: HttpServletRequest) = params.getAsOrElse("skip", 0)

get("/things/:id") {
  implicit val request = this.request
  Future {
    params("id")  // no more NPE
  }
}

post("/things") {
  implicit val response = this.response
  myThingsActor ? Post(parsedBody.extract[Things]) map { things =>
    if (things.isEmpty) status = 404 // no more NPE
    ()
  }
}

// assuming scentry is mixed in and user is something stored on the request or in cookies or something
get("/stuff/:id") {
  implicit val request = this.request
  implicit val response = this.response
  val stuff: Future[Stuff] = getStuff(params("id"))

  stuff map { allTheThings =>
    getTrinketsForUser(allTheThings, user, skip)  // no more NPE
  }
}
```

With the AsyncResult you get another chance to add some default context to your async operations but other than that it works very similar.

```scala

def skip(implicit request: HttpServletRequest) = params.getAsOrElse("skip", 0)

get("/things/:id") {
  new AsyncResult { val is =
    Future {
      params("id")  // no more NPE
    }
  }
}

post("/things") {
  new AsyncResult { val is =
    myThingsActor ? Post(parsedBody.extract[Things]) map { things =>
      if (things.isEmpty) status = 404 // no more NPE
      ()
    }
  }
}

// assuming scentry is mixed in and user is something stored on the request or in cookies or something
get("/stuff/:id") {
  new AsyncResult { val is = {
    val stuff: Future[Stuff] = getStuff(params("id"))

    stuff map { allTheThings =>
      getTrinketsForUser(allTheThings, user, skip)  // no more NPE
    }
  } }
}
```

The AsyncResult has an implicit parameter of `ScalatraContext` and every `ScalatraBase` has an implicit conversion to a `ScalatraContext` so the request and response are now stable values and no longer stuck in thread-locals.
With that bug out of the way, and you're a swagger user then the next examples are for you.

#### New swagger API

In the previous version of scalatra we introduced swagger support. While the API we introduced then worked it ended up being very messy and was error prone since most of it used strings. At wordnik we started using scalatra and one of my co-workers, who just started learning scalatra, remarked: Swagger makes Scalatra ugly. Clearly something had to be done about this!
This release tries to fix some of that by using as much information from the context as it possibly can and defining a fluent api for describing swagger operations.

There are no more strings except for things that are notes, descriptions, names etc. It integrates with scalatra's commands so you only define the parameters for a request once. It automatically registers models when you provide them and it converts the scalatra route matcher to a swagger path string. Let's take a look at a before and after:

This is how it used to be:

```scala
// declare the models, and the models it uses
// case class Pet(id: Long, category: Category, name: String, urls: List[String], tags: List[Tag], status: String)
models = Map("Pet" -> classOf[Pet], "Category" -> classOf[Category], "Tag" -> classOf[Tag])

// declare the route with the swagger annotations
get("/findByStatus",
  summary("Finds Pets by status"),
  nickname("findPetsByStatus"),
  responseClass("List[Pet]"),
  endpoint("findByStatus"),
  notes("Multiple status values can be provided with comma separated strings"),
  parameters(
    Parameter("status",
      "Status values that need to be considered for filter",
      DataType.String,
      paramType = ParamType.Query,
      defaultValue = Some("available"),
      allowableValues = AllowableValues("available", "pending", "sold")))) {
  data.findPetsByStatus(params("status"))  // this is our actual implementaton, you might have missed it.
}
```

This is what it is now:

```scala
// declare the swagger operation description
val findByStatus =
  (apiOperation[List[Pet]]("findPetsByStatus")
    summary "Finds Pets by status"
    notes "Multiple status values can be provided with comma separated strings"
    parameter (queryParam[String]("status").required  // required is the default value so not strictly necessary
                description "Status values that need to be considered for filter"
                defaultValue "available"
                allowableValues ("available", "pending", "sold")))

// declare the route with the swagger annotation
get("/findByStatus", operation(findByStatus)) {
  data.findPetsByStatus(params("status"))
}
```

So there is no more endpoint declaration necessary, you work with actual types and you don't have to remember to register models and all their referenced models anymore. In my opinion if you simply write the swagger declaration close to where your route lives you still have the docs live with the code but it won't obscure the application code anymore.

Let me know what you think.
