# mwan3-sticky-flush

Removes the **sticky** assignments of a failed WAN from mwan3, so clients leave through the other WAN right away instead of waiting for the timeout (600 s by default).

## The problem

With `sticky=1` on an mwan3 rule, each source IP is assigned to a WAN for `timeout` seconds. If that WAN goes down, mwan3 stops using it for new traffic, but the sticky entries still point to it. The clients on that IP stay without internet until the entry expires.

This block deletes those entries as soon as the WAN goes down.

## How it works

mwan3 stores sticky state in an ipset `mwan3_rule_ipv4_<rule>` of type `hash:ip,mark`. Each entry is `IP,0x<mark>`, where the WAN mark is `mwan3_get_iface_id << 8` in 8-digit hex (for example `0x00000100` for the first WAN and `0x00000200` for the second).

On `ifdown` (or on `disconnected` when the WAN does not answer to ping), the block walks the `mwan3_rule_ipv4_*` sets and deletes the entries with the failed WAN's mark. Entries for the other WAN are not touched.

If `disconnected` arrives but the WAN still answers to ping (for example, when mwan3 restarts), nothing is deleted.

## Installation

1. Back up your configuration:
   ```sh
   cp /etc/mwan3.user /etc/mwan3.user.bak
   ```
2. Append the contents of `mwan3-sticky-flush.user` to the end of `/etc/mwan3.user`:
   ```sh
   cat mwan3-sticky-flush.user >> /etc/mwan3.user
   sh -n /etc/mwan3.user && echo "syntax OK"
   ```
3. No mwan3 restart is needed: `/etc/mwan3.user` is read on every interface event.

Requirements: `ipset` (already a mwan3 dependency when sticky is used). `conntrack` is not needed.

## Verification

To check without touching live traffic, use scratch sets: copy the block, replace `mwan3_rule_ipv4_` with a test prefix, create a set with entries for your WAN's mark, and run the block with `ACTION=ifdown INTERFACE=wan`.

On a real failure you should see this in the log:

```sh
logread | grep mwan3-sticky-flush
# sticky entries of wan removed (mark 0x00000100)
```

And the number of entries with the failed WAN's mark should drop to 0:

```sh
ipset save mwan3_rule_ipv4_<rule> | grep -c ",0x00000100"
```

## Tested on

- GL.iNet GL-MT6000 (Flint-2), OpenWrt 25.12.6, mwan3 2.x, two WANs (one DHCP, one PPPoE).
- Real WAN failure with `ifdown` and recovery with `ifup`: the failed WAN's entries went from 1 to 0 in about 3 seconds, and new traffic left through the other WAN.
- Isolated tests with scratch sets for the `ifdown` case and for `disconnected` with the WAN still up.

Not tested on other OpenWrt versions, other hardware, or setups with more than two WANs.

## Limitations

- IPv4 only (`mwan3_rule_ipv4_*`). IPv6 entries are not touched.
- Relies on mwan3 internals (set names and the `mwan3_get_iface_id` function). If a future version changes them, the block needs to be reviewed.
- It removes sticky entries, not open connections. Connections that already existed through the failed WAN keep their mark until mwan3's connection flush cleans them up, or until they close.

## Credits

The selective connection flush this approach is based on comes from [vjt/mwan3-selective-flush](https://github.com/vjt/mwan3-selective-flush) (MIT). This version removes sticky entries from the ipset instead of conntrack connections.

## License

MIT, see [LICENSE](LICENSE).
