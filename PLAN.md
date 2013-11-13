This repo hopes to be the service provisioner for backend services for data-backed components.

This service will have a privileged relationship with the Appmaker app (via a shared secret).  People who want to run their own Appmaker instance can run an instance of Haas as well.

Haas's job is to provision Hoodie endpoints.  Hoodie-using components will be configured by the Appmaker designer with a Hoodie endpoint (HEP) provisioned by Haas, but if someone wants to configure their Appmaker component to use a distinct Hoodie endpoint, that's fine too.

Haas's architecture is to have an HTTP endpoint to provision a new hoodie instance, and then to be a multi-domain proxy which routes HTTP requests to the right hoodie instance.


```


                    +------------------------------------------------------------------------------+
                    |                                         +--------------------------------+   |
                    |  +-----------------------+              | docker container for app 1     |   |
       /provision   |  | - given username,app, |              |--------------------------------|   |
       +----------->|  |   email               |         5521 |                                |   |
                    |  | - provision docker co.|   +----------+ 80   - runs hoodie-server node |   |
                    |  | - register in db      |   |          |        process                 |   |
                    |  |                       |   |          |      - runs couchdb or pouchdb |   |
                    |  |                       |   |          |      - can migrate to shared…  |   |
                    |  +-----------------------+   |          +--------------------------------+   |
                    |                              |                                               |
                    |                              |          +--------------------------------+   |
                    |                              |          | docker container for app 2     |   |
                    | +------------------------------+        |--------------------------------|   |
       per-hoodie   | | domain-to-unique-port-mapper |   5522 |      … couchdb in the future   |   |
      +------------>| | (e.g. HAProxy)               |--------+ 80   once segmentation is      |   |
       calls e.g.   | |                              |        |      ready.                    |   |
                    | +-+----------------------------+        |                                |   |
  myapp.haas.moz.net|   |  +--------------------+   |         |                                |   |
                    |   +--+  ship manifest     |   |         +--------------------------------+   |
                    |      |--------------------|   |                                              |
                    |      | a simple db that   |   |         +--------------------------------+   |
                    |      | maps app.username  |   |         | docker container for app 3     |   |
                    |      | to docker instance |   |    5523 |--------------------------------|   |
                    |      | and port #         |   +---------+ 80                             |   |
                    |      +--------------------+             |                                |   |
                    +-----------------------------------------+--------------------------------+---+
```

The `/provision` endpoint sends a secret in the payload which is pre-arranged, so that haas only accepts provisioning requests from known parties.  `/provision` requires in addition a triplet of (`username`, `emailaddress`, and `appname`).  Given these three, a container named (`<username>-<app>`) is spun up, and the container is told which ports to use to map to the ports hoodie provides.

TODO: figure out how the hoodie-server process and/or the couchdb instance are told that the 'admin' is the user, as we effectively need SSO from the appmaker designer through to the hoodie/couch processes.

The "ship manifest" database is told to record the mapping between (`username`, `app`) and either docker names or ports.

The "domain-to-unique-port-mapper" listens to http requests on a wildcard domain (e.g. `*.hoodies.moz.whatever`), and maps (`app.user.subdomain:80`) requests to exist-only-on-the-container-ship ports which docker creates, and which map inside the containers to `:80` or whatever port hoodie talks to.  Jan says that HAProxy supports enough of the HTTP spec bits that Hoodie relies on that it can work well.

TODO: figure out who gets to clean up unused containers, policy therefore, and whether there's a /deprovision API.
