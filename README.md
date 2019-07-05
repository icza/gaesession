# gaesession

[![GoDoc](https://godoc.org/github.com/icza/gaesession?status.svg)](https://godoc.org/github.com/icza/gaesession)

Google App Engine (GAE) support for https://github.com/icza/session.

The implementation stores sessions in the Memcache and also saves sessions in the Datastore as a backup
in case data would be removed from the Memcache. This behaviour is optional, Datastore can be disabled completely.
You can also choose whether saving to Datastore happens synchronously (in the same goroutine)
or asynchronously (in another goroutine), resulting in faster response times.

We can use `NewMemcacheStore()` and `NewMemcacheStoreOptions()` functions to create a session Store implementation
which stores sessions in GAE's Memcache. Important to note that since accessing the Memcache relies on
Appengine Context which is bound to an `http.Request`, the returned Store can only be used for the lifetime of a request!
Note that the Store will automatically "flush" sessions accessed from it when the Store is closed,
so it is very important to close the Store at the end of your request; this is usually done by closing
the session manager to which you passed the store (preferably with the defer statement).

So in each request handling we have to create a new session manager using a new Store, and we can use the session manager
to do session-related tasks, something like this:

    ctx := appengine.NewContext(r)
    sessmgr := session.NewCookieManager(gaesession.NewMemcacheStore(ctx))
    defer sessmgr.Close() // This will ensure changes made to the session are auto-saved
                          // in Memcache (and optionally in the Datastore).

    sess := sessmgr.Get(r) // Get current session
    if sess != nil {
        // Session exists, do something with it.
        ctx.Infof("Count: %v", sess.Attr("Count"))
    } else {
        // No session yet, let's create one and add it:
        sess = session.NewSession()
        sess.SetAttr("Count", 1)
        sessmgr.Add(sess, w)
    }

Expired sessions are not automatically removed from the Datastore. To remove expired sessions, the package
provides a `PurgeExpiredSessFromDSFunc()` function which returns an [http.HandlerFunc](https://golang.org/pkg/net/http/#HandlerFunc).
It is recommended to register the returned handler function to a path which then can be defined
as a cron job to be called periodically, e.g. in every 30 minutes or so (your choice).
As cron handlers may run up to 10 minutes, the returned handler will stop at 8 minutes
to complete safely even if there are more expired, undeleted sessions.
It can be registered like this:

    http.HandleFunc("/demo/purge", gaesession.PurgeExpiredSessFromDSFunc(""))

Check out the GAE session demo application which shows how it can be used.
[cron.yaml](https://github.com/icza/gaesession/blob/master/gae_session_demo/cron.yaml) file of the demo shows how a cron job can be defined to purge expired sessions.

Check out the [GAE session demo application](https://github.com/icza/gaesession/blob/master/_gae_session_demo/gae_session_demo.go) which shows how to use this in action.
