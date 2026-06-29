#The Overview row
Now the panels. I'll do the first stat and the first gauge in full clicks, then the repeats just call out what changes, since the mechanics are identical.
Click Add → Visualization. You're in the panel editor: query editor at the bottom, visualization picker and options on the right.
Uptime (stat):

##Top-right visualization dropdown → Stat.
In the query editor, confirm the datasource is ${datasource}, make sure you're on the Code tab (not Builder), and paste:

   time() - node_boot_time_seconds{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job"}

Expand the query's Options row → set Legend to Custom → {{instance}}.
Right pane → Standard options → Unit → search "seconds" → pick seconds (s). It'll render as days/hours.
Panel title Uptime. Back to dashboard.

That's the whole loop for every panel: pick viz → paste query → set legend → set unit → title. Repeat with these:
CPU Cores (stat), unit short:
count by (instance) (count by (instance, cpu) (node_cpu_seconds_total{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job",mode="idle"}))
Total RAM (stat), unit bytes (IEC):
node_memory_MemTotal_bytes{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job"}
CPU Busy (gauge): this one introduces the gauge type. Pick Gauge, paste the query, then in the right pane set Standard options → Min = 0, Max = 100, Unit = percent (0-100). Then open the Thresholds section, switch to Absolute, and add steps: green at base, yellow at 70, red at 90. Those thresholds are what color the arc.
100 - (avg by (instance) (rate(node_cpu_seconds_total{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job",mode="idle"}[$__rate_interval])) * 100)
Memory Used (gauge): duplicate the CPU Busy panel (panel menu → More → Duplicate) so you inherit the thresholds and min/max, then just swap the query:
100 * (1 - (node_memory_MemAvailable_bytes{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job"} / node_memory_MemTotal_bytes{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job"}))
Now drag the five panels into a row and resize: the three stats narrow on the left, the two gauges wider on the right. To label the band, click Add → Row, name it Overview, and drag it above these panels.
###4. The CPU row
These are your first timeseries panels, and they have a couple of options the stats don't.
CPU Usage by Mode: Add → Visualization → Time series. Query:
avg by (instance, mode) (rate(node_cpu_seconds_total{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job",mode!="idle"}[$__rate_interval])) * 100
Legend → Custom → {{instance}} - {{mode}}. Unit = percent (0-100), Min = 0. Then the timeseries-specific bits in the right pane: under Graph styles set Fill opacity to ~20 and, importantly, Stack series → Normal — that's what makes the modes pile into a total-busy height. Under Legend, set Mode: Table and Placement: Bottom, and tick the Mean/Max/Last calcs so you get the summary columns.
Load Average: Time series, three queries (click + Query twice to get A/B/C):
node_load1{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job"}
node_load5{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job"}
node_load15{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job"}
Legends {{instance}} load1 / load5 / load15. Unit short, Min 0, Fill opacity 0 (these are lines, not areas), no stacking.
Wrap both in a CPU row.
5. Memory, Disk, Network rows
Same timeseries mechanics; here's purely what differs per panel.
Memory Used % — unit percent (0-100), fill ~15, no stacking:
100 * (1 - (node_memory_MemAvailable_bytes{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job"} / node_memory_MemTotal_bytes{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job"}))
Swap Used — unit bytes (IEC), Min 0:
node_memory_SwapTotal_bytes{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job"} - node_memory_SwapFree_bytes{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job"}
Filesystem Used % — unit percent (0-100), fill 0, legend {{instance}}:{{mountpoint}}. Note the filter moves inside both metric selectors:
100 - (node_filesystem_avail_bytes{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job",fstype!~"tmpfs|squashfs|overlay|ramfs|devtmpfs|fuse.*|iso9660"} * 100 / node_filesystem_size_bytes{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job",fstype!~"tmpfs|squashfs|overlay|ramfs|devtmpfs|fuse.*|iso9660"})
Disk I/O — unit bytes/sec (IEC) (Bps), two queries:
rate(node_disk_read_bytes_total{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job",device!~"dm-.*|loop.*|sr.*|ram.*"}[$__rate_interval])
rate(node_disk_written_bytes_total{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job",device!~"dm-.*|loop.*|sr.*|ram.*"}[$__rate_interval])
Legends {{instance}}:{{device}} read / write.
Network Traffic — unit bits/sec (SI) (bps), full-width, two queries:
rate(node_network_receive_bytes_total{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job",device!~"lo|veth.*|docker.*|br-.*|cni.*|flannel.*|virbr.*"}[$__rate_interval]) * 8
rate(node_network_transmit_bytes_total{tenant=~"$tenant",site=~"$site",instance=~"$instance",job=~"$job",device!~"lo|veth.*|docker.*|br-.*|cni.*|flannel.*|virbr.*"}[$__rate_interval]) * 8
Legends {{instance}}:{{device}} rx / tx.
6. Finish
Set the dashboard time range to now-1h, auto-refresh to 30s (top-right controls), then Save. Pick a tenant in the dropdown and confirm the site/job/instance lists narrow accordingly — that's the cascade proving itself.
A couple of habits that'll save you grief as you go: build one panel of each type carefully (one stat, one gauge, one timeseries) and then Duplicate rather than rebuild — you inherit units, thresholds, and legend settings and only swap the query, which is faster and keeps styling consistent. And whenever a rate panel looks flat or empty, your first suspect is the same $__rate_interval/scrape-interval relationship from the SNMP work, not the query itself.
