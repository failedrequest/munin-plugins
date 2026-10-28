# munin-plugins
This is a collection of FreeBSD munin plugins. They have been tested to work with minimunin and may work on Solaris and Illumos with some minor changes.

All plugins live in [`FreeBSD/`](FreeBSD/). Plugins whose name ends in `_` are wildcard plugins: symlink them into the munin plugin directory with the argument appended to the name (for example `arp_em0`).

| Plugin | Language | Category | Wildcard argument |
|--------|----------|----------|-------------------|
| [`arp_`](FreeBSD/arp_) | sh | network | interface name |
| [`ifss_`](FreeBSD/ifss_) | python3 | network | interface name |
| [`ipss_`](FreeBSD/ipss_) | python3 | network | interface name |
| [`ktls_new_sessions`](FreeBSD/ktls_new_sessions) | sh | nginx | none |
| [`ktls_records`](FreeBSD/ktls_records) | sh | nginx | none |
| [`ktls_sessions`](FreeBSD/ktls_sessions) | sh | nginx | none |
| [`nginx_connections`](FreeBSD/nginx_connections) | sh | nginx | none |
| [`nginx_requests`](FreeBSD/nginx_requests) | sh | nginx | none |
| [`tcp-states`](FreeBSD/tcp-states) | sh | network | none |
| [`weather_`](FreeBSD/weather_) | python3 | sensors | airport code / location |

## arp_
Graphs the number of entries in the ARP table for one interface.

- Runs `arp -na -i <interface>` and counts the lines returned.
- Single gauge: `total` (ARP Total).
- Example: `ln -s /path/to/arp_ /usr/local/etc/munin/plugins/arp_em0`

## ifss_
"Interface SuperSample": graphs traffic (bits per second) on one interface, sampled every 30 seconds instead of munin's default 5 minutes.

- Link it as `ifss_<interface>`, for example `ifss_em0`. Nothing else needs configuring.
- The first munin poll starts a background collector by itself. It runs `netstat --libxo json -ibn -I <interface>` every 30 seconds and queues a timestamp plus the input and output byte counters.
- Each poll hands munin every queued sample with its original timestamp, and the queue is emptied without losing or repeating samples.
- Only one collector runs per interface (`flock` on a pid file). Running `ifss_em0 acquire` by hand, or from an old rc script, is harmless: it exits at once if a collector is already running.
- The collector gives up after 20 failed samples in a row (about 10 minutes), for example if the interface disappears. The next poll starts it again.
- The queue is capped at about 64 KB, so an unpolled plugin cannot fill the disk.
- State files (`ifss_<if>.pid`, `.spool`, `.read`) go in `$MUNIN_PLUGSTATE`, or `/tmp` if it is not set. They are mode 0600 and symlinks are refused.
- The interface name is validated, and `netstat` is run without a shell.
- Graph shows received traffic below the axis (-) and sent traffic above it (+), converted from bytes to bits.
- Uses `update_rate 30` and a custom RRD layout (`30s for 1w, 1m for 1t, 5m for 1y`).
- Upgrading from the old version: delete the leftover `/tmp/<interface>-stats.db` and stop any old `ifss_<if> acquire` process or rc entry. The new plugin does not use them.

## ipss_
"IP SuperSample": same design as `ifss_`, but graphs packets per second instead of bits per second.

- Link it as `ipss_<interface>`, for example `ipss_em0`. The first munin poll starts the background collector by itself; see `ifss_` for the details of how the collector, queue and state files work (the files are named `ipss_<if>.*`).
- Samples the input and output packet counters from `netstat --libxo json -ibn -I <interface>` every 30 seconds.
- Graph shows received packets (-) and sent packets (+). No bytes-to-bits conversion is applied.
- Upgrading from the old version: delete the leftover `/tmp/<interface>-pps-stats.db` and stop any old `ipss_<if> acquire` process or rc entry.

## ktls_new_sessions
Graphs how often the kernel creates new KTLS (kernel TLS) sessions, plus its error counters. One session is created per TLS connection handed to the kernel, so this is the KTLS connection rate.

- Reads `kern.ipc.tls.stats.*` with `sysctl -n`. All seven fields are `DERIVE` counters with `min 0`, so they are graphed per second.
- Fields: `enable_calls` (attempts, drawn as an area), `offload_total` (successful), `failed_crypto`, `switch_failed`, `switch_to_sw`, `switch_to_ifnet`, `corrupted_records`.
- A counter that cannot be read is reported as `U` (unknown).
- `autoconf` succeeds when `kern.ipc.tls.stats.enable_calls` exists.
- Example: `ln -s /path/to/ktls_new_sessions /usr/local/etc/minimunin-plugins/ktls_new_sessions`

## ktls_records
Graphs TLS records the kernel encrypts and decrypts per second, by TLS version and cipher (the software/OCF path).

- Reads `kern.ipc.tls.stats.ocf.*` with `sysctl -n`. All fields are `DERIVE` counters with `min 0`.
- Eleven fields: encrypts for `tls13_gcm`, `tls12_gcm`, `tls13_chacha20`, `tls12_chacha20`, `tls11_cbc` and `tls10_cbc`, and decrypts for `tls13_gcm`, `tls12_gcm`, `tls13_chacha20`, `tls12_chacha20` and `tls11_cbc`. The encrypt fields are stacked.
- A counter that cannot be read is reported as `U`.
- `autoconf` succeeds when `kern.ipc.tls.stats.ocf.tls13_gcm_encrypts` exists.

## ktls_sessions
Graphs the number of active KTLS sessions, by mode and cipher.

- Reads `kern.ipc.tls.{sw,ifnet,toe}.{gcm,chacha20,cbc}` with `sysctl -n`. These are `GAUGE`s. The three `sw.*` fields are stacked.
- Field names use `_` instead of `.` (for example `sw_gcm`), because munin field names cannot contain a dot.
- Adds a `total` line from `kern.ipc.tls.stats.active`.
- A value that cannot be read is reported as `U`.
- `autoconf` succeeds when `kern.ipc.tls.enable` exists.

## nginx_connections
Graphs nginx connections by state, from nginx's `stub_status` page.

- Fetches the status page with `/usr/bin/fetch -q -T 5` (5 second timeout). The default URL is `http://127.0.0.1:8080/nginx_status`; set `NGINX_STATUS_URL` to use another.
- Needs a `stub_status` location in nginx that serves that URL.
- Four `GAUGE`s: `active`, `reading`, `writing`, `waiting`.
- `autoconf` succeeds when the page can be read and starts with `Active connections`.

## nginx_requests
Graphs nginx requests and connections per second, from the same `stub_status` page.

- Same URL, `NGINX_STATUS_URL` override and `fetch` call as `nginx_connections`.
- Three `DERIVE` counters with `min 0`: `requests` (drawn as an area), `accepts` (accepted connections) and `handled` (handled connections). The values come from the third line of the page.
- `autoconf` succeeds when the page can be read and starts with `Active connections`.

## tcp-states
Graphs the number of TCP connections in each state across all interfaces.

- Parses `netstat -an -p tcp -f inet` and counts connections per state, including `LISTEN`.
- States use the FreeBSD names: `ESTABLISHED`, `SYN_SENT`, `SYN_RCVD`, `FIN_WAIT_1`, `FIN_WAIT_2`, `TIME_WAIT`, `CLOSED`, `CLOSE_WAIT`, `LAST_ACK`, `CLOSING`, `LISTEN`.
- `autoconf` succeeds when `netstat -an -p tcp -f inet` runs.

## weather_
Graphs current weather for a location using data from [wttr.in](https://wttr.in).

- The location comes from the plugin name suffix (for example `weather_KBOS`) or from the `code` environment variable.
- Fetches `http://wttr.in/<code>?format=j1` and reports the current temperature (F), "feels like" temperature (F), and humidity.
- Requires outbound HTTP access from the munin node.
