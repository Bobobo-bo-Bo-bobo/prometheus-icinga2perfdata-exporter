**__Note__**: Because I'm running my own servers for serveral years, all data and repositories are hosted there
at <https://git.ypbind.de/cgit/prometheus-icinga2perfdata-exporter/about/>

---

Export performance data reported from Icinga2 checks to Prometheus
==================================================================

## Preface
[Icinga2](https://www.icinga.com/products/icinga-2/) ships with some writer modules for processing performance data reported by checks:

  * generic [PerfdataWriter](https://github.com/Icinga/icinga2/blob/master/doc/09-object-types.md#objecttype-perfdatawriter) placing performance data in a directory for futher processing (e.g. suitable for [PNP4Nagios](://docs.pnp4nagios.org/))
  * [Graphite](https://graphite.readthedocs.io/en/latest/) via [GraphiteWriter](https://github.com/Icinga/icinga2/blob/master/doc/09-object-types.md#objecttype-graphitewriter)
  * [InfluxDB](https://www.influxdata.com/time-series-platform/influxdb/) via [InfluxdbWriter](https://github.com/Icinga/icinga2/blob/master/doc/09-object-types.md#objecttype-influxdbwriter)

For several reasons some sites run [Prometheus](https://prometheus.io) to collectd and display several performance statistics, so it would be nice to
have the reported performance data available Prometheus too.

---

## Method of operation
[prometheus-icinga2perfdata-exporter](https://git.ypbind.de/cgit/prometheus-icinga2perfdata-exporter/) runs as a process in the foreground (suitable for running as [systemd](https://www.freedesktop.org/wiki/Software/systemd/))
service.

It connects to the [Icinga2 EventStream API](https://www.icinga.com/docs/icinga2/latest/doc/12-icinga2-api/#event-streams) API of an Icinga2 instance and listens for [`CheckResult`](https://www.icinga.com/docs/icinga2/latest/doc/12-icinga2-api/#event-stream-types)
events.

The performance data reported in the `CheckResult` objects are parsed and stored in memory. [prometheus-icinga2perfdata-exporter](https://git.ypbind.de/cgit/prometheus-icinga2perfdata-exporter/) also listens (default port 19997 on all interfaces) for
HTTP connections and reports the collected data when `/metrics` is queried by the Prometheus scraper.

### Normalizing unit of measurements
According the [Nagios Plugins Development Guidelines](https://nagios-plugins.org/doc/guidelines.html#AEN200) unit of measurement can be only one of:

  * no unit
  * s - seconds (also us, ms)
  * % - percentage
  * B - bytes (also KB, MB, TB)
  * c - a continous counter 

Those units are recognized and scaled to their base units as suggested in the [Metric and label naming best practices] (https://prometheus.io/docs/practices/naming/#base-units) for Prometheus:

  * no unit - not scaled
  * s - `ms`, `us`, ... are scaled to seconds
  * % - scaled to 0 ... 1
  * B - `KB`, ... are scaled to byte; multiple of _1024_ byte (assumed as `KiB`, `GiB`, to be precise)
  * c - not scaled

---

## Build requirements
### Go!
Obviously because the exporter has been written in [Go](https://golang.org/)

### Go package - github.com/go-ini/ini
Parsing the INI format of configuration is done using the [github.com/go-ini/ini](https://github.com/go-ini/ini) package

### Go package - github.com/nu7hatch/gouuid
The [Icinga2 EventStream API](https://www.icinga.com/docs/icinga2/latest/doc/12-icinga2-api/#event-streams) requires an unique name for the event queue.

This is implemented by using UUID version 4 (specified in [RFC 4122: A Universally Unique IDentifier (UUID) URN Namespace](https://www.ietf.org/rfc/rfc4122.txt))
of the [github.com/nu7hatch/gouuid](https://github.com/nu7hatch/gouuid) pakage

---

## Configuration
### Icinga2 permissions
[prometheus-icinga2perfdata-exporter](https://git.ypbind.de/cgit/prometheus-icinga2perfdata-exporter/) collects the performance data by connecting to the Icinga2 EventStream API and listen to
[`CheckResult`](https://www.icinga.com/docs/icinga2/latest/doc/12-icinga2-api/#event-stream-types) types.

Therefore the only required permission for the API user used to connect is `events/checkresult`. 

### prometheus-icinga2perfdata-exporter configuration
At the moment two sections are recognized.

The mandatory `icinga2` section specify the connection and authentication parameters to connect to the Icinga2 API and the
optional `exporter` section allows for overriding the instance name and for expiration timer for collected performance data.

#### Section `icinga2`
Mandatory is the `[icinga2]` section as it contains the connection and authentication options required to connect to the EventStream API of Icinga2.

Parameters are:

  * `host` - host name (or IP address) of the Icinga2 instance to connect to
  * `port` - port number to connect to; _Default:_ 5665 (note: this *must* be a number, service names are not supported)
  * `method` - authentication method, this can be either
    * `user` - authenticate with a user/password combination using HTTP basic authentication
    * `cert` - authenticate using a X.509 certificate
  * `user` - username to use for authentication if `method` is set to `user`
  * `password`- password to use for authentication if method is set to `password`
  * `cert_file` - absolute path to the public key of the X.509 certificate used for authentication if `method` is set to `cert`
  * `key_file` - absolute path to the private key of the X.509 certificate used for authentication if `method` is set to `cert` (note: due to the lack of Go to load encrypted keys only _unencrypted_ X.509 keys are supported)
  * `ca_cert` - absolute path to the CA file for the Icinga2 service
  * `insecure_ssl` - skip verification if the hostname provided by the certificate of the Icinga2 instance matches the `host` name, valid values are `true` or `false` (default)

#### Section `exporter`
The optional section `exporter` supports two options:

  * `instance` - override the name of the reporting instance (label `instance`), the default is the `host:port` of the Icinga2 instance connected to
  * `expiration` - expiration time for collected performance data in seconds (see below)

An `expiration` threshold is disabled by default as there is no "sane" default.

Setting the threshold is a double-edged sword.

It prevents reporting of stale performance data not updated or no longer reported by checks (e.g. if a check has been removed).

If used it should be set to a multiple (to accommodate for the spread of scheduled checks by Icinga2) of the *longest* check interval.
Setting the value to low will evict valid performance data of checks scheduled with a long interval.

When the expiration is set a separate Go routine will scan the collected performance data at the configured interval and remove all date
older than `expiration` seconds. This routine starts immediately after [prometheus-icinga2perfdata-exporter](https://git.ypbind.de/cgit/prometheus-icinga2perfdata-exporter/)
has been started.

If time jump is detected (the timestamp of the performance data as reported in the `CheckResult` object is in the future) the data will _not_ removed.

#### Sample configuration

`[icinga2]`

`host = icinga2.server`

`port = 5665`

`; authentication method to use`

`; user - user and password`

`; cert - client certificate

`method = cert`


`; for user method`

`; user = icinga2-api-user`

`# password = secret password`


`; for "cert" method`

`cert_file = /path/to/client/cert.pem`

`key_file = /path/to/client/cert.key`

`; CA certificate`

`; ca_cert =`

`ca_cert = /path/to/ca/used/to/sign/icinga2/ap/cert.pem`


`; skip certificate verification`

`insecure_ssl = false`


`[exporter]`

`instance = override_instance_name`

`; expiration time for metrics in seconds`

`; expiration = 3600`

---

## Command line parameters
prometheus-icinga2perfdata-exporter supports the following command line parameters:

  * `-config-file` - location of the configuration file, default: `/etc/prometheus/prometheus-icinga2perfdata-exporter.ini`
  * `-help` - displays a short help text
  * `-listen-address` - address to listen for requests, default: `:19997`

---

## License
This program is licenses under [GLPv3](http://www.gnu.org/copyleft/gpl.html).


