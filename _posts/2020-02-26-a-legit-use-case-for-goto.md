---
title: A legit use case for the goto abomination
layout: post
tags:
    - c
    - programming
keywords:
    - c
    - goto
---

Most programming educational material mentions, at some point, the
`goto` keyword, usually as a despicable abomination that should not
exist in the face of this earth. The general consensus is that the
[goto statement is considered
harmful](http://www.u.arizona.edu/~rubinson/copyright_violations/Go_To_Considered_Harmful.html)
and the advice is to avoid the use of `goto` and to restructure the
code so as to eliminate the need for using it. Of course this is sound
advice, intended to discourage the creation of code in which a
function's control flow is heavily influenced by jumps to and from
various points within its body, making and debugging really difficult.

Such an evil construct should surely be banished by all languages by
now. However, some widely-used languages offer the goto statement in
their arsenal. These include C, C++, Golang, and C#, while Java
interestingly has a goto statement but [it is not
implemented](https://stackoverflow.com/a/4547764). However, most
dynamic languages (ruby, python, javascript) do not have a goto
statement (although [this was a fun
read](http://patshaughnessy.net/2012/2/29/the-joke-is-on-us-how-ruby-1-9-supports-the-goto-statement)!).

I had never used the goto statement myself until I started writing C
code for a certain kind of embedded device and encountered multiple
situations where I wish I had some kind of try/catch/finally construct
in the language. The purpose of the latter is to better express the
need for ensuring that a bunch of statements will run no matter what
especially on early-exit error paths. Now, C of course does not have
such facilities but does have `goto`. Can we use it to handle such use
cases in a C program in a way that is clean and DRY?

Let's see a (somewhat contrived) example: suppose that we have to
parse a json string of the form `"{"name": "John"}` in a C program
using the well-known library
[cJSON](https://github.com/DaveGamble/cJSON). For this purpose, we
define a `handler` function (that maybe validates and aggregates the
attributes in some way - it doesn't really matter) and a
`process_json` function that parses the json string and calls
`handler` for each attribute that is parsed, as follows:

```C
// return 0 for success and -1 for error
int handler(char *key, char *val) {
    // some condition gives an error
    if (key == NULL || val == NULL) { return -1; }
    // do something here
    return 0;
}

// process the supplied json string
// returns 0 on success, -1 on error
int process_json(const char *json, ) {
    int rc = 0;
    // here we allocate all the resources
    // root will need to be freed further down
    cJSON *root = cJSON_Parse(json);
    // let's grab a reference to the name item
    cJSON *name = cJSON_GetObjectItem(root, "name");
    // and `handle` the inner string
    if ((rc = handler("name", cJSON_GetStringValue(name))) < 0) {
        // stop processing - we have an error
        // free the resource
        cJSON_Delete(root);
        return rc;
    }
    // happy path --  free the resource
    cJSON_Delete(root);
    return rc;
}
```

Now, the above code works fine but it is a little bit repetitive since
we have to free the `root` resource in two exit paths. There could
also be more paths (e.g. for json attributes to parse) and more
resources that are created in the flow and that need to be released
before exit. All this repetition is error-prone and makes our function
less robust and more succeptible to leaks.

Let's see how we can use the `goto` statement in order to concentrate
all of our clean-up code in one place and reduce the likelihood of a
leak. The idea is to use an `end` label and drive error exit paths to
that label using `goto` statements as follows:

```C
int process_json(const char *json) {
    int rc = 0;
    cJSON *root = cJSON_Parse(json);
    cJSON *name = cJSON_GetObjectItem(root, "name");
    if ((rc = handler("name", cJSON_GetStringValue(name))) < 0) { goto end; }

    // ... other code here / attribute parsing etc. ...

    end:
    // free the resource
    cJSON_Delete(root);
    return rc;
}
```

In the above flow, we will reach the `end` label either by going
through the whole function (happy path) or by exiting prematurely when
an error occurs. This way, (a) there is no repetition, (b) we ensure
that the cleanup will occur no matter what and, (c) we improve the
readability of the function's flow by reducing the visual statement
clutter. In short, We have a cleaner and safer function.

So, when implementing a C function with the following
characteristics/constaints:

* error conditions are handled before the happy path, and
* one or more resources need to be cleaned up upon exit regardless of
  premature/error or normal exit,

then the use of `goto` is quite legitimate and results in cleaner,
safer, DRYer code.
