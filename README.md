# munin-plugins
This is a collection of FreeBSD munin plugins. They have been tested to work with minimunin and may work on Solaris and Illumos with some minor changes.

All plugins live in [`FreeBSD/`](FreeBSD/). Plugins whose name ends in `_` are wildcard plugins: symlink them into the munin plugin directory with the argument appended to the name (for example `arp_em0`).

| Plugin | Language | Category | Wildcard argument |
|--------|----------|----------|-------------------|
| [`arp_`](FreeBSD/arp_) | sh | network | interface name |
| [`ifss_`](FreeBSD/ifss_) | python3 | network | interface name |
| [`ipss_`](FreeBSD/ipss_) | python3 | network | interface name |
| [`tcp-states`](FreeBSD/tcp-states) | sh | network | none |
| [`weather_`](FreeBSD/weather_) | python3.7 | sensors | airport code / location |

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
- Note that the shebang is `python3.7`; adjust it if that interpreter is not installed.

## Testing and debugging

- `python3 -m unittest discover -s tests` runs the unit tests (for `ifss_` and `ipss_`).
- `tools/plugin-run <plugin> [instance] [command]` runs a plugin the way munin would, using a throw-away state directory, and checks the output format. Examples: `tools/plugin-run tcp-states`, `tools/plugin-run ifss_ em0 config`.
