# Nerves.WpaSupplicant
[![Build Status](https://travis-ci.org/nerves-project/nerves_wpa_supplicant.svg?branch=master)](https://travis-ci.org/nerves-project/nerves_wpa_supplicant)
[![Hex version](https://img.shields.io/hexpm/v/nerves_wpa_supplicant.svg "Hex version")](https://hex.pm/packages/nerves_wpa_supplicant)

This package enables Elixir applications to query or effect changes on Wi-Fi
network connections of a host system, by interacting with a [wireless
supplicant].

The portable [wpa_supplicant] daemon handles Wi-Fi operations like scanning for
wireless networks, connecting, authenticating, and collecting wireless adapter
statistics. `Nerves.WpaSupplicant` uses the control interface provided by this
supplicant implementation to bring these capabilities to your Elixir code.

*This library is under development and we plan on making the API more user-friendly in future versions.*

## Installation

1. Add `nerves_wpa_supplicant` to your list of dependencies in `mix.exs`:

        def deps do
          [{:nerves_wpa_supplicant, "~> 0.2.1"}]
        end

2. Ensure `nerves_wpa_supplicant` is started before your application:

        def application do
          [applications: [:nerves_wpa_supplicant]]
        end

## Note on permissions

The `wpa_supplicant` daemon runs as root and requires processes that attach to
its control interface to be root. This project contains a C port process—called
`wpa_ex`—whose sole purpose is to interact with the `wpa_supplicant` daemon,
but it needs sufficient permission to do so. The `Makefile` contains logic to
mark `wpa_ex` setuid root so that this works, but you may want to change this
depending on your setup.

## Building

Building `nerves_wpa_supplicant` is similar to other Elixir projects. The
`Makefile` will invoke `mix` to compile both the Elixir and C source code. The
only additional step is to ensure that permissions are suitable on the `wpa_ex`
binary as described in the preceding section. You'll be asked for your password
for this step when you run `make`, by default.

    $ make

If you want to disable the setuid root step in the Makefile, just set the `SUDO`
environment variable to `true` to make it a no-op:

    $ SUDO=true make

If you need to use a different askpass program, you can set that as well:

    $ SUDO_ASKPASS=/usr/bin/ssh-askpass make

## Running

The `wpa_supplicant` daemon must be running already on your system and the control
interface must be exposed. If you have any doubt, try running `wpa_cli`. If that
doesn't work, the Elixir `Nerves.WpaSupplicant` won't work.

If you're on a system where you can start the `wpa_supplicant` manually, here's
an example command line:

    $ /sbin/wpa_supplicant -iwlan0 -C/var/run/wpa_supplicant -B

Once you're happy that the `wpa_supplicant` is running, start `iex` by running:

    $ iex -S mix

Start a `Nerves.WpaSupplicant` process:

    iex> {:ok, pid} = Nerves.WpaSupplicant.start_link("/var/run/wpa_supplicant/wlan0")
    {:ok, #PID<0.82.0>}

You can sanity-check that Elixir has properly attached to the `wpa_supplicant`
daemon by pinging the daemon:

    iex> Nerves.WpaSupplicant.request(pid, :PING)
    :PONG

To scan for access points, call `Nerves.WpaSupplicant.scan/1`. This can take a few
seconds:

    iex> Nerves.WpaSupplicant.scan(pid)
    [%{age: 42, beacon_int: 100, bssid: "00:1f:90:db:45:54", capabilities: 1073,
       flags: "[WEP][ESS]", freq: 2462, id: 8,
       ie: "00053153555434010882848b0c1296182403010b07",
       level: -83, noise: 0, qual: 0, ssid: "1SUT4", tsf: 580579066269},
     %{age: 109, beacon_int: 100, bssid: "00:18:39:7a:23:e8", capabilities: 1041,
       flags: "[WEP][ESS]", freq: 2412, id: 5,
       ie: "00076c696e6b737973010882848b962430486c0301",
       level: -86, noise: 0, qual: 0, ssid: "linksys", tsf: 464957892243},
     %{age: 42, beacon_int: 100, bssid: "1c:7e:e5:32:d1:f8", capabilities: 1041,
       flags: "[WPA2-PSK-CCMP][ESS]", freq: 2412, id: 0,
       ie: "000768756e6c657468010882848b960c1218240301",
       level: -43, noise: 0, qual: 0, ssid: "dlink", tsf: 580587711245}]

To attach to an access point, you need to configure a network entry in the
`wpa_supplicant`. The `wpa_supplicant` can have multiple network entries
configured. The following removes all network entries so that only one is
configured:

    iex> Nerves.WpaSupplicant.set_network(pid, ssid: "MyNetworkSsid", key_mgmt: :WPA_PSK, psk: "secret")
    :ok

If the access point is around, the `wpa_supplicant` will eventually connect to
the network.

    iex> Nerves.WpaSupplicant.status(pid)
    %{address: "84:3a:4b:11:95:23", bssid: "1c:7e:e5:32:de:32",
      group_cipher: "TKIP", id: 0, key_mgmt: "WPA2-PSK", mode: "station",
      pairwise_cipher: "CCMP", ssid: "MyNetworkSsid", wpa_state: "COMPLETED"}

Polling the `wpa_supplicant` for status isn't ideal, so it's possible to
register a `GenEvent` with `Nerves.WpaSupplicant`. If you don't supply one with
a call to `start_link/2`, one is automatically created and available via
`Nerves.WpaSupplicant.event_manager/1`. The following example shows how to view
events at the prompt:

    iex> defmodule Forwarder do
    ...>  use GenEvent
    ...>  def handle_event(event, parent) do
    ...>    send parent, event
    ...>    {:ok, parent}
    ...>  end
    ...> end
    iex> Nerves.WpaSupplicant.event_manager(pid) |> GenEvent.add_handler(Forwarder, self())
    iex> flush
    {:nerves_wpa_supplicant, #PID<0.85.0>, :"CTRL-EVENT-SCAN-STARTED"}
    {:nerves_wpa_supplicant, #PID<0.85.0>, :"CTRL-EVENT-SCAN-RESULTS"}

## Low-level messaging

It is expected that the helper functions for interacting with the `wpa_supplicant`
will not cover every situation. The `Nerves.WpaSupplicant.request/2` function allows
you to send arbitrary commands. Requests are atoms that are named the same as
described in the `wpa_supplicant` documentation (see *Useful links*). If a request
takes a parameter, pass it as a tuple where the first element is the command.
Parameters may be strings or numbers and will be properly formatted for the
control interface. The response is also parsed and turned into atoms, numbers,
strings, lists, or maps depending on the command. The string parsing is taken
care of by this library. Here are some examples:

    iex> Nerves.WpaSupplicant.request(pid, :INTERFACES)
    ["wlan0"]

    iex> Nerves.WpaSupplicant.request(pid, {:GET_NETWORK, 0, :key_mgmt})
    "WPA-PSK"

## Useful links

  1. [API Reference](https://hexdocs.pm/nerves_wpa_supplicant/api-reference.html)
  2. [wpa_supplicant homepage][wpa_supplicant]
  3. [wpa_supplicant control interface](http://w1.fi/wpa_supplicant/devel/ctrl_iface_page.html)
  4. [wpa_supplicant information on the archlinux wiki](https://wiki.archlinux.org/index.php/Wpa_supplicant)

## Licensing

The majority of this package is licensed under the Apache 2.0 license. The code
that directly interfaces with the `wpa_supplicant` is copied from the
`wpa_supplicant` package and has the following copyright and license:

```
/*
 * wpa_supplicant/hostapd control interface library
 * Copyright (c) 2004-2007, Jouni Malinen <j@w1.fi>
 *
 * This software may be distributed under the terms of the BSD license.
 * See README for more details.
 */
```

[wireless supplicant]: https://en.wikipedia.org/wiki/Wireless_supplicant
[wpa_supplicant]: http://w1.fi/wpa_supplicant/
