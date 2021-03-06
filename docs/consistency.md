# Djangae Contrib Consistency

A contrib app which helps to mitigate against eventual consistency issues with the App Engine Datastore.

**Note: If all you want to do is make sure that a newly updated/created object is returned as part of a queryset
take a look at [djangae.db.consistency.ensure_instance_included](db_backend.md#djangaedbconsistencyensure_instance_included).
If you want a powerful solution to general consistency issues then this is the app for you!**

## In A Nutshell

It caches recently created and/or modified objects so that it knows about them even if they're not
yet being returned by the Datastore.  It then provides a function `improve_queryset_consistency`
which you can use on a queryset so that it:

* Uses the cache to include recently-created/-modified objects which match the query but which are
  not yet being returned by the Datastore. (This is not effective if the cache gets purged).
* Does not return any recently deleted objects which are still being returned by the Datastore.
  (This is not cache-dependent.)


## Basic Usage

* Add `'djangae.contrib.consistency'` to `settings.INSTALLED_APPS` (if you want the tests to run).
* Add `from djangae.contrib.consistency.signals import connect_signals; connect_signals()` somewhere in your setup code (see **Notes** below).

```python

from consistency import improve_queryset_consistency, get_recent_objects

queryset = MyModel.objects.filter(is_yellow=True)
more_consistent_queryset = improve_queryset_consistency(queryset)
# Use as normal
```

If you want you can use the `get_recent_objects` function separately to just get hold of the recently-created/modified objects.


Note that the recently-created objects are not guaranteed to be returned.  The
recently-created objects are only stored in memcache, which could be purged at any time.


## Advanced Configuration

By default the app will cache recently-created objects for all models.  But you can change this
behaviour so that it only caches particular models, only caches objects that match particular
criteria, and/or caches objects that were recently *modified* as well as recently *created*.

```python

# settings.py

CONSISTENCY_CONFIG = {

    # These defaults apply to every model, unless otherwise overriden.
    # The values shown here are the default defaults (i.e. what you get if
    # you don't set CONSISTENCY_CONFIG at all).
    "defaults": {
        "cache_on_creation": True,
        "cache_on_modification": False,
        "cache_time": 60, # Seconds
        "caches": ["django"], # Where to store the cache.
        "only_cache_matching": [], # Optional filtering (see below)
    },

    # The settings can be overridden for each individual model
    "models": {
        "app_name.modelname": {
            "cache_on_creation": True,
            "cache_on_modification": True,
            "caches": ["session", "django"],
            "cache_time": 20,
            "only_cache_matching": [
                # A list of checks, where each check is a dict of filter
                # kwargs or a function. If an object matches *any* of
                # these then it is cached. No filters means cache everything.
                {"name": "Ted", "archived": False},
                lambda obj: obj.method(),
            ]
        },
        "app_name.unimportantmodel": {
            "cache_on_creation": False,
            "cache_on_modification": False,
            # Any settings which you don't override inherit from "defaults".
        },
    },
}
```


## Notes

* Even if you set both `cache_on_creation` and `cache_on_modification` to `False`, you can still use
  `improve_queryset_consistency` to prevent stale objects from being returned by your query.
* The way that `improve_queryset_consistency` works means that it converts your query into a
  `pk__in` query.  This has 2 side effects:
    - It causes the initial query to be executed.  It does this with `values_list('pk')` (which
      becomes a Datastore keys-only query) so is fast, but you should be aware that it hits the DB.
    - It introduces a limit of 1000 results, so if your queryset is not already limited then a
      limit will be imposed, and if your queryset already has a limit then it may be reduced. This
      is to ensure a total result of <= 1000 objects.  This is imperfect though, and may result in
      slightly fewer than 1000 results because recent objects in the cache will reduce the limit
      even if they don't match the query. (This could potentially be fixed.)
* To avoid the side effects of `improve_queryset_consistency` you may wish to use
  `get_recent_objects` instead, giving you slightly more control over what happens.
* As you can see in the config section, there are 2 places which the new objects can be cached: in
  Django's normal cache (`django.core.cache`) or in the current user's session.
    - Using the "session" cache may be slightly faster for querying (as the session object has probably
      been loaded anyway, so it avoids another cache lookup), but it's unlikely to be faster when
      creating/modifying an object, because writing to the session requires a Database write, which is
      probably slower than a cache write.  Unless you're altering the session object anyway, in which
      case the session cache may be advantageous.
* We deliberately don't register these signals automatically (either in the app's models files or in
  the app's AppConfig) because registering signals causes Django's bulk delete behaviour to change
  which in turn causes some of the core Django tests (which are run as part of the Djangae
  'testapp') to fail.  If you can find a way around this then send a pull request!
