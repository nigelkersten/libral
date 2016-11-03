# Libral

Libral is a sketch of an attempt to try and build a native provider
platform/framework. The goals of this are

* Roughly match the functionality of the `puppet resource` command
* Serve as the basis for Puppet's provider system
* Avoid being Puppet specific as much as possible to make it useful for
  other uses
* Make writing providers very easy

## Required packages

You will need to install [Boost](http://boost.org) for program_options and
to use many of the libraries in
[Leatherman](https://github.com/puppetlabs/leatherman). You will also need
[Augeas](http://augeas.net/)

## Todo list

- [X] finish mount provider
- [ ] add a shell provider
- [ ] add a remote provider (using an HTTP API)
- [ ] adapt providers to multiple OS (maybe using mount)
- [ ] more core providers
- [ ] noop mode
- [ ] event reporting

## Some language

Naming things is hard; here's the terms libral uses:

* _Type_: the abstract description of an entity we need to manage; this is
  the external interface through which entities are managed. I am not
  entirely convinced this belongs in libral at all
* _Provider_: something that knows how to manage a certain kind of thing
  with very specific means; for example something that knows how to manage
  users with the `user*` commands, or how to manage mounts on Linux
* _Resource_: an instance of something that a provider manages. This is
  closely tied both to what is being managed and how it is being
  managed. The important thing is that resources expose a desired-state
  interface and therefore abstract away the details of how changes are made

**FIXME**: we need some conventions around some special resource
properties; especially, namevar should always be 'name' and the primary key
for all resources from this provider, and 'ensure' should have a special
meaning (or should it?)

### Open questions
- Do we need types at all at this level ?
- Can we get away without any provider selection logic ? There are two
  reasons why provider selection is necessary:
  * adjust to system differences. Could we push those to compile time ?
  * manage different things that are somewhat similar, like system packages
    and gems, or local users and LDAP users ? This would push this
    modelling decision into a layer above libral.
- Wouldn't it be better to make providers responsible for `noop` mode
  rather than making it part of the framework ?
- Would it be better to make providers responsible for event generation
  rather than doing that in the framework ?

## Provider lifecycle

The intent is that the lifecycle for providers will be roughly something
like this:

```cpp
    some_provider prov();

    if (prov.suitable()) {
      // initialize provider internals
      prov.prepare();

      // Loop over all resources
      for (auto rsrc : prov.instances()) {
        auto should = get_new_attrs_from_somewhere();
        rsrc.update(should);
      }

      // or do something to a specific resource
      auto rsrc = prov.find(some_name);
      rsrc.destroy();

      // or create a new one
      auto rsrc = prov.create("new_one");
      auto should = get_new_attrs_from_somewhere_else();
      rsrc.update(should);
      rsrc.flush()

      // make sure all changes have been written to disk
      prov.flush();

      // not sure yet if we need an explicit 'close' call
      prov.close()
    }
```

* Rather than returning just a `bool`, at some point `suitable()` will need
  to return more details about what the provider does and whether it can be
  used
* At some point, we'll need a notion of a context that tells providers
  about system details (like facts) and some settings; would be cool to use
  that to change the idea of where the root of the FS is, for example.

## External providers

What resources `libral` can manage is determined by what providers are
available. Some providers are built in and implemented in C++, but doing
that is of course labor-intensive and should only be done for good
reason. It is much simpler, and recommended, that new providers first be
implemented as external providers. External providers are nothing more than
scripts or other executables that follow one of `libral`'s calling
conventions. The different calling conventions trade off implementation
complexity for expressive power.

The following calling conventions are available. If you are just getting
started with `libral`, you should write your first providers using hte
`simple` calling convention.

* [simple](doc/invoke-simple.md)
* `augeas` (planned): when you mostly need to twiddle entries in a file,
and maybe run a command
* `ansible` (planned): use your Ansible 2 modules as an external provider
* `json` (planned,maybe): input/output via JSON, with the possibility of
  batching operations
