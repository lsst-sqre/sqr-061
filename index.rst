..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Services and infrastructure underlying the RSP need to be instrumented to allow effective monitoring of performance and usage. This technote proposes the architectural approach for doing so. 

Appendix: Monitoring targets
============================

A preliminary list of things that we want to monitor.
This includes both potential alerts and potential performance metrics to understand overall trends.

Metrics
-------

As much as possible (and where there is an associated user), the following metrics should be tagged with the authenticated user so that we can see metrics by user or trace a 500 error back to a specific user.

For each web application:

- Healthy / not healthy that ideally runs a synthetic transaction through any underlying database
- Counters for each HTTP status
- Usage counters for each route, as identified by the application (not from the URL, since that will probably generate too many too-specific routes)
- Performance buckets (p50, p90, p99) by route
- Unique users

Kubernetes
^^^^^^^^^^

- CPU usage over time for pods, tagged with Argo CD application and separating out user notebook pods
- Memory usage over time for pods, tagged with Argo CD application
- API failures to the control plane by status code over time

Gafaelfawr
^^^^^^^^^^

- Internal and notebook token cache hits and misses
- New token creation counts by token type over time
- Total number of non-expired sessions
- Total number of non-expired user tokens
- Remaining token lifetime of requests, tagged with user and token type
- Login failures
- Admin actions taken

Image cutouts
^^^^^^^^^^^^^

- Sync requests (tagged with success or failure)
- Async requests (tagged with success or failure)
- Duration of processing for the request

mobu
^^^^

- Healthy / not healthy over time for each service
- Success/failure of each attempt, broken down by step for those probes that have multiple steps (such as notebook tests)
- Number of probes (so that we know if mobu is testing that service at all)
- Timing information for all meaningful timers (i.e., not idles or intentional delays)

nublado
^^^^^^^

- Number of labs (ideally divided between active and idle)
- Number of lab spawns over time

TAP
^^^

- Number of sync and async TAP queries over time
- Time required to complete a TAP query
- Number of failed TAP queries

Alerts
------

The following alerts are good/bad alerts akin to Nagios probes rather than metric-based alerts.
Many of the metrics may also have useful associated threshold alerts, which are not discussed in this section.

These alerts should translate into Slack alerts when they fire.

- External inaccessibility of a public-facing site
- Remaining TLS certificate lifetime for each important public-facing site less than some threshold (30 days?)
- Remaining Kubernetes control plane TLS certificate lifetime less than some threshold for every Kubernetes cluster (will require per-cluster monitoring infrastructure or some agent in each cluster that calls out to the monitoring system, due to firewalls)
- Remaining lifetime of the tokens for our Vault service accounts
- Argo CD applications in failed state
- cachemachine failure to pre-pull images after more than some threshold of time
- cert-manager fails to refresh a desired certificate
- vault-secrets-operator fails to refresh a desired secret
- neophile processing failed or produced errors on for a package

We will also want some general infrastructure to run a validation script and have it report any findings to Slack.
It may make sense to implement the above using the same mechanism.
Example uses:

- List all Route 53 DNS entries for IP addresses not controlled by Rubin
- List all unexpected Kubernetes objects in a cluster (not part of Kubernetes itself and not managed by Argo CD)
- Unexpected errors in the logs of some application (retrieved from Google's log aggregator, for instance)

Finally, we should probably have Slack alerts for some interesting events:

- Vault secret creation
- Vault secret deletion
- Administrative cluster actions taken (notifying Slack when we sync an app in the production environment, for example)
- Gafaelfawr token administrative actions taken, with exceptions for known routine cases such as mobu

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
