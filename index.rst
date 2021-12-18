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

As much as possible, the following metrics should be tagged with the authenticated user so that we can see metrics by user or trace a 500 error back to a specific user.

For each web application:

- Healthy / not healthy that ideally runs a synthetic transaction through any underlying database
- Counters for each HTTP status
- Usage counters for each route, as identified by the application (not from the URL, since that will probably generate too many too-specific routes)
- Performance buckets (p50, p90, p99) by route
- Unique users

From mobu:

- Healthy / not healthy over time for each service
- Success percentage over time (may be the same as healthy / not healthy)
- Number of probes (so that we know if mobu is testing that service at all)
- Performance buckets (p50, p90, p99) for the meaningful timers (i.e., not idles or intentional delays) for that monkey

Gafaelfawr
^^^^^^^^^^

- Internal and notebook token cache hits and misses
- New token creation counts by token type over time
- Remaining token lifetime of requests, tagged with user and token type
- Login failures
- Admin actions taken

Image cutouts
^^^^^^^^^^^^^

- Sync requests (tagged with success or failure)
- Async requests (tagged with success or failure)
- Performance metrics (p50, p90, p99) for cutout creation (this isn't retrievable from route metrics for async jobs)

Alerts
------

The following alerts are good/bad alerts akin to Nagios probes rather than metric-based alerts.
They should translate into Slack alerts when they fire.

- Remaining TLS certificate lifetime for each important public-facing site less than some threshold (30 days?)
- Remaining Kubernetes control plane TLS certificate lifetime less than some threshold for every Kubernetes cluster (will require per-cluster monitoring infrastructure or some agent in each cluster that calls out to the monitoring system, due to firewalls)
- Remaining lifetime of the tokens for our Vault service accounts
- Argo CD applications in failed state

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

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
