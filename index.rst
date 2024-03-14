###################################
Monitoring architecture for the RSP
###################################

.. abstract::

   Services and infrastructure underlying the RSP need to be instrumented to allow effective monitoring of performance and usage. This technote proposes the architectural approach for doing so. 

Monitoring Server Implementation: Current state
===============================================

The server-side implementation of monitoring is held in `Roundtable
<https://github.com/lsst-sqre/roundtable>`_.  It can be found in the
`deployments/monitoring
<https://github.com/lsst-sqre/roundtable/tree/master/deployments/monitoring>`_
directory.

Influx
------

The most important component of it is the InfluxDBv2 database, which
collects all the telegraf-monitored data from the Rubin ecosystem.  It
is accessible at https://monitoring.lsst.codes.

This Influx database presents dashboards for users and generates Slack
alerts when problems arise.

Bucket Nomenclature
```````````````````

Because we are using https://monitoring.lsst.codes to monitor only
Phalanx and Roundtable services, we have settled on a simple
nomenclature.  Any bucket that begins with a leading underscore is a
system bucket, and we can't do anything about it.  This is an Influx
choice, not a Rubin choice.  That leaves two other categories of
buckets.

- Buckets with a name that does not end in an underscore are presumed to
  be application buckets, collecting data for the service of the same
  name (and, generally, from the same namespace), e.g. ``nublado2`` or
  ``sasquatch``.  These buckets all have a one-week expiration period.

- Any bucket with a name that ends in an underscore is an internal
  bucket.  Currently we use ``multiapp_`` to capture measurements across
  multiple clusters and applications for which we should alert, and
  ``alerted_`` to mark an event as already having generated an alert and
  thus not needing another.  Currently ``roundtable_internal_`` and
  ``roundtable_prometheus_`` are also in use for monitoring roundtable
  itself.  In the future, when we have collapsed roundtable and phalanx
  into a single repository, we will no longer need these and we will
  treat roundtable applications as simply other services.

This design decision does imply one bucket per application.  Since all
these buckets have the same retention period, this may be inefficient,
particularly since all the ones collecting Kubernetes metrics have the
same shape.  We have not yet seen any performance problems result from
this choice, but 20 buckets is generally considered to be a good upper
limit, and we are already well above that.  Thus we may start
aggregating all of our Kubernetes metrics, at least, into a single
bucket.

Buckets should never need to be created manually.  Buckets and tasks are
instead checked against the phalanx configuration periodically, and a
Kubernetes cronjob creates any missing ones.  That logic will be
explained in the next section.

Rubin-Influx-Tools
------------------

The actual configuration for the database is held in a different
repository: `Rubin Influx Tools
<https://github.com/lsst-sqre/rubin-influx-tools/>`_.  This repository
contains most of the server-side configuration for our Influx
installation.  A brief guide to its contents follows.

Jobs
````

`src/rubin_influx_tools
<https://github.com/lsst-sqre/rubin-influx-tools/tree/main/src/rubin_influx_tools>`_
contains the Python implementations that we use to automatically track
application buckets and create alert tasks for them.  The general idea
is that as new applications are added to Phalanx, the ``bucketmaker``
cronjob will notice and create a bucket for data pertaining to the new
app.  In general, each cronjob runs every fifteen minutes although their
start times are staggered.  These jobs can also be run manually, and can
be provided with a ``FORCE`` setting to destroy and recreate existing
objects.  This is especially useful for ``taskmaker`` in that the common
case of changing one thing about a particular query means it needs to be
changed in several dozen tasks.


Bucketmaker
:::::::::::

Bucketmaker is run periodically as a K8s Cronjob.  It starts up, checks
out the Phalanx repository, and parses the science-platform application
definitions to determine whether there are any new services for which it
doesn't have buckets.  If there are, it creates those buckets.

Bucketmapper
::::::::::::

For boring technical reasons (explained below) the general user
interface to InfluxDBv2 in our use-case is through Chronograf.
Chronograf only knows about InfluxDBv1, and therefore an accomodation
layer, which maps each bucket to a database retention policy (DBRP) is
needed.  Bucketmapper examines its application buckets and creates a
corresponding DBRP if necessary.  It is also a K8s Cronjob.

Taskmaker
:::::::::

The final Kubernetes Cronjob manages alerting tasks.

The various application buckets all have similar sets of tasks attached:
periodically, the task will scan recent bucket entries for interesting
data, and write its findings to some output database where they will be
used as inputs for Slack alerts.

This (as long as we keep a one-bucket-per-application model) is best
done by templating the tasks and iterating over our applications.
Taskmaker does exactly this, creating any missing tasks as needed, and
then creating the task for each category of exciting event that consumes
alertable tasks from an input bucket (usually ``multiapp_``), sends a
Slack alert, and records the alert as having been sent (in the
``alerted_`` bucket).  These buckets are created as part of the
``taskmaker`` run if they are not present.  They have the shortest
retention period InfluxDB allows, which is 1 hour: they are intended to
hold ephemeral data only as long as needed to send an alert via Slack.

Note that if we dispense with application buckets, ``multiapp_``, whose
purpose is to aggregate information across applications, becomes
unnecessary.

Tokenmaker
::::::::::

Tokenmaker is not a cron job; rather it is something you run once when
setting up the installation.  It uses the actual administrative token
(created as part of the initial InfluxDBv2 install) to generate both a
bucket read/write token (which is the token used by all the satellite
telegraf instances feeding data into InfluxDB) and the taskmaker token,
which has many more capabilities, but still falls short of being full
admin, even for the organization that owns all the buckets
(``square``, in our case).

Dashboards
``````````

It has so far been our experience that automated creation of dashboards
is extremely fiddly.  Thus, at the moment, dashboards are not
automatically created.  If you reinstall the central monitoring server,
instead you can just restore the `dashboards
<https://github.com/lsst-sqre/rubin-influx-tools/tree/main/src/rubin_influx_tools/dashboards>`_
from the json stored in the directories there.  Note that there are
separate dashboards for InfluxDBv2 and Chronograf.  These are intended
to be identical in terms of function, but the necessary specifications
change.  Ideally, some day, InfluxDBv2 has a broader set of
authentication methods available and we can dispense with needing
Chronograf and its configuration entirely.  Today is not that day.

Templates
`````````

The query templates used by `taskmaker
<https://github.com/lsst-sqre/rubin-influx-tools/blob/main/src/rubin_influx_tools/taskmaker.py>`_
are found in
`templates
<https://github.com/lsst-sqre/rubin-influx-tools/tree/main/src/rubin_influx_tools/templates>`_.
These use Jinja2 templating, but anything not inside double moustaches
is plain old Flux script.  Note that there's no good way to create an
importable library of Flux functions without rebuilding Flux itself, so
many helper functions are duplicated, especially between the Slack
alerting tasks.  A more-carefully designed templating process could
implement some sort of preprocessing step to import functions into the
scripts.


Influx
------

We are using InfluxDBv2 as our central collection database, and we are
using the standard helm chart from
https://github.com/influxdata/helm-charts/tree/master/charts/influxdb2 .

Configuration for this chart (and for Chronograf) is in `values.yaml <https://github.com/lsst-sqre/roundtable/blob/master/deployments/monitoring/values.yaml>`_ .

This is a very close-to-stock installation of InfluxDBv2.  The only
strange thing we do is provide our own ingress resource so that we can
mount chronograf at a subpath.

Currently we are using a 20GiB maximum database size.  This can be
increased if necessary, but at the moment, given our data rates and a
7-day rentention policy, it seems to be sufficient.

Chronograf
----------

Since we have decided on InfluxDBv2, why do we need Chronograf at all?
After all, InfluxDB has at least as good a UI with version 2.  The
answer is both dumb and sad.  InfluxDBv2 only has token authentication,
with manually-created tokens.  This will get unwieldy very fast.
Chronograf, on the other hand, allows authenticating through an Oauth2
connector, so we can base authorization on GitHub or Google group
membership.

That's the entire reason we need Chronograf; even its dashboards are
just the Chronograf version of the InfluxDBv2 dashboards.

Telegraf-ds
-----------

The eagle-eyed reader of ``values.yaml`` will have noticed a
``telegraf-ds`` configuration as well.  This is basically a version of
the telegraf-ds application found client-side with each RSP instance,
and we will discuss it there.  The difference here is that this
configuration is used for monitoring InfluxDBv2 itself (and therefore
K8s applications in the ``roundtable`` cluster).

Monitoring Client Implementation: Current state
===============================================

Each RSP instance has both a ``telegraf`` and a ``telegraf-ds``
application.  Each of these is nothing more than a set of ``telegraf``
processes that feeds data back to the central Influx database.

However, their purposes, and the data they collect, is quite different.

These reside `within Phalanx
<https://github.com/lsst-sqre/phalanx/tree/master/services>`_.

Telegraf
--------

We are currently using Telegraf to monitor prometheus endpoints for the
services that expose them.  Each RSP has a single telegraf process to
scrape all prometheus endpoints.

This is probably less than ideal.  We would like to move to
``telegraf-operator`` for this task, but at the moment it will not allow
destruction of namespaces, nor will it allow the JupyterHub Kubespawner
to create user lab pods.  I have not yet dug into the operator source to
determine how easy this is to correct, but there's an open `issue
<https://github.com/influxdata/telegraf-operator/issues/81>`_ to address
better namespace handling, so I have piled on with our use case.

`Telegraf's values.yaml <https://github.com/lsst-sqre/phalanx/blob/master/services/telegraf/values.yaml>`_
contains the definition of each application and the endpoints it exposes
in its ``prometheus_config`` section.  When you add a new service which
provides prometheus endpoints, you must update the telegraf config to
know about it.  When deployed, Helm templating handles creation of the
appropriate telegraf config to scrape each endpoint.

The prometheus data is used to populate the ``ArgoCD``, ``HTTP
Requests``, and ``JupyterLab Servers`` dashboards.

Telegraf-ds
-----------

Telegraf-ds, on the other hand, is used to collect Kubernetes-specific
data from each application.  The `configmap template <https://github.com/lsst-sqre/phalanx/blob/master/services/telegraf-ds/templates/configmap.yaml>`_
provides a Helm template that collects CPU and memory statistics from
application pods, as well as pod restart and state statistics.  These
are fed into the application buckets in the central InfluxDBv2 server, and
from there exciting data populates the ``multiapp_`` bucket to generate
alerts.  The application buckets themselves are used to populate the ``K8s
Applications`` dashboards in InfluxDB and Chronograf.

That's why it's telegraf-ds: it runs as a DaemonSet (one telegraf
process per node in each cluster) and collects local statistics from
each Kubernetes node, and then splits that information by application
for delivery to InfluxDBv2.

The telegraf-ds implementation on Roundtable populates the ``Roundtable
InfluxDB2`` dashboard, which contains information about overall disk
pressure, point-write success rates, and storage shard size (by
application) within InfluxDBv2.

Compliance with identified monitoring targets
=============================================

Our current system, while useful, does not actually implement a huge
fraction of the things we want to track.

The "Metrics" goals are spotty at best.  All we've really implemented is
the HTTP response counters (as exposed by ingress-nginx).

We have mostly hit the "Kubernetes" goals: memory and CPU usage over
time are tracked.  K8s API problems are not directly exposed, but if
those issues cause a pod to restart or go into a non-running state, that
will trigger an alert.

The "JupyterLab Servers" dashboard gives us the number of
currently-running JupyterLab servers, but does not give us any
information about how heavily utilized those servers actually are.  We
would need to spawn a telegraf sidecar with the JupyterLab pods in order
to collect that data, which is not impossible, but probably should wait
until our (large) remote spawner work.

Appendix: Monitoring targets
============================

A preliminary list of things that we want to monitor.
This includes both potential alerts and potential performance metrics to understand overall trends.

Metrics
-------

As much as possible (and where there is an associated user), the following metrics should be tagged with the authenticated user so that we can see metrics by user or trace a 500 error back to a specific user.

For each web application:

- Healthy / not healthy that ideally runs a synthetic transaction through any underlying database
- Counters or events for each HTTP status
- Usage counters for each route, as identified by the application (not from the URL, since that will probably generate too many too-specific routes) for a specific period
- Time required to respond to each request by route
- User agents/ calling service

Kubernetes
``````````

- CPU usage over time for pods, tagged with Argo CD application and separating out user notebook pods
- Memory usage over time for pods, tagged with Argo CD application
- API failures to the control plane by status code over time
- OOM kill statistics [nublado-users]

Gafaelfawr
``````````

- Internal and notebook token cache hits and misses
- LDAP data cache hits and misses
- New token creation counts by token type over time
- Total number of non-expired sessions
- Total number of non-expired user tokens
- Remaining token lifetime of requests, tagged with user and token type
- OpenID Connect authentications by registered OIDC client
- Login failures
- Admin actions taken
- Distribution of "not seen since" times (and which users) (that would result in insights like "50% of users have not logged onto any services in the last 6 months")
- Any user statistics per user group [eg project, science user]
- Overall traffic by user and service, as input to rate limiting decisions
- Rate limit rejections by user

Image cutouts (or similar)
``````````````````````````

- Sync requests (tagged with success or failure)
- Async requests (tagged with success or failure)
- Duration of processing for the request
- Unique authenticated non-bot users in the past year, 90 days, 30 days, 7 days, and day
- Distribution of area sizes for the requested cutouts (returned size as a fraction of original size, or area of requested cutout)
- Data type for basis of operation (PVI, coadd etc)
- User frequency / long tail chart over a period (leads to insights like "70% of users have never used this service")

mobu
````

- Healthy / not healthy over time for each service
- Success/failure of each attempt, broken down by step for those probes that have multiple steps (such as notebook tests)
- Number of probes (so that we know if mobu is testing that service at all)
- Timing information for all meaningful timers (i.e., not idles or intentional delays)

Nublado
```````

- Number of labs (ideally divided between active and idle) (now & timeseries)
- Number of lab spawns over time
- Unique authenticated non-bot users in the past year, 90 days, 30 days, 7 days, and day
- User frequency / long tail chart over a period (leads to insights like "70% of users have never used this service but this one user hammers it 24/7")
- Frequency of use of any JupyterLab plugins
- Number of running file servers (now & timeseries)
- Number of file server spawns over time
- Labs culled by the idle culler
- Labs terminated because their maximum run time expired

Storage
```````
- Size of files in home space, distribution

Portal
``````

- Usage counts for which portions of the Portal users are using
- Unique authenticated non-bot users in the past year, 90 days, 30 days, 7 days, and day

TAP (or similar)
````````````````

- Number of sync and async TAP queries over time
- Time required to complete a TAP query (distribution)
- Time spent in service (after Qserv returns)
- Number of failed TAP queries
- Unique authenticated non-bot users in the past year, 90 days, 30 days, 7 days, and day
- User frequency / long tail chart over a period (leads to insights like "70% of users have never used this service")
- Size of returned results (distribution)
- All of the above "per endpoint" for multiple endpoints (Qserv, Postgres)
- Invoking user agent (portal, pyvo)

Sasquatch/Influx
````````````````

- Number of topics
- Frequency of data per topic
- Volume rate per topic and overall
- Query load
- Queries and queries by user
- Latency data

Alerts
``````

The following alerts are good/bad alerts akin to Nagios probes rather than metric-based alerts.
Many of the metrics may also have useful associated threshold alerts, which are not discussed in this section.

These alerts should translate into Slack alerts when they fire.

General deployment alerts
`````````````````````````

- External inaccessibility of a public-facing site
- Remaining TLS certificate lifetime for each important public-facing site less than some threshold (30 days?)
- Remaining Kubernetes control plane TLS certificate lifetime less than some threshold for every Kubernetes cluster (will require per-cluster monitoring infrastructure or some agent in each cluster that calls out to the monitoring system, due to firewalls)
- Remaining lifetime of the tokens for our Vault service accounts
- Argo CD applications in failed state

Specific applications
`````````````````````

- cachemachine failure to pre-pull images after more than some threshold of time
- cert-manager fails to refresh a desired certificate
- vault-secrets-operator fails to refresh a desired secret
- neophile processing failed or produced errors on for a package

Validation alerts
`````````````````

We will also want some general infrastructure to run a validation script and have it report any findings to Slack.
It may make sense to implement the above using the same mechanism.
Example uses:

- List all Route 53 DNS entries for IP addresses not controlled by Rubin
- List all unexpected Kubernetes objects in a cluster (not part of Kubernetes itself and not managed by Argo CD)
- Unexpected errors in the logs of some application (retrieved from Google's log aggregator, for instance)

Interesting events
``````````````````

Finally, we should probably have Slack alerts for some interesting events:

- Vault secret creation
- Vault secret deletion
- Administrative cluster actions taken (notifying Slack when we sync an app in the production environment, for example)
- Gafaelfawr token administrative actions taken, with exceptions for known routine cases such as mobu
