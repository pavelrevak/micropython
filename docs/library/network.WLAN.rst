.. currentmodule:: network
.. _network.WLAN:

class WLAN -- control built-in WiFi interfaces
==============================================

This class provides a driver for WiFi network processors.  Example usage::

    import network
    # enable station interface and connect to WiFi access point
    nic = network.WLAN(network.WLAN.IF_STA)
    nic.active(True)
    nic.connect('your-ssid', 'your-key')
    # now use sockets as usual

Constructors
------------
.. class:: WLAN(interface_id)

Create a WLAN network interface object. Supported interfaces are
``network.WLAN.IF_STA`` (station aka client, connects to upstream WiFi access
points) and ``network.WLAN.IF_AP`` (access point, allows other WiFi clients to
connect). Availability of the methods below depends on interface type.
For example, only STA interface may `WLAN.connect()` to an access point.

Methods
-------

.. method:: WLAN.active([is_active])

    Activate ("up") or deactivate ("down") network interface, if boolean
    argument is passed. Otherwise, query current state if no argument is
    provided. Most other methods require active interface.

.. method:: WLAN.connect(ssid=None, key=None, *, bssid=None)

    Connect to the specified wireless network, using the specified key.
    If *bssid* is given then the connection will be restricted to the
    access-point with that MAC address (the *ssid* must also be specified
    in this case).

.. method:: WLAN.disconnect()

    Disconnect from the currently connected wireless network.

.. method:: WLAN.scan()

    Scan for the available wireless networks.
    Hidden networks -- where the SSID is not broadcast -- will also be scanned
    if the WLAN interface allows it.

    Scanning is only possible on STA interface. Returns list of tuples with
    the information about WiFi access points:

        (ssid, bssid, channel, RSSI, security, hidden)

    *bssid* is hardware address of an access point, in binary form, returned as
    bytes object. You can use `binascii.hexlify()` to convert it to ASCII form.

    There are five values for security:

        * 0 -- open
        * 1 -- WEP
        * 2 -- WPA-PSK
        * 3 -- WPA2-PSK
        * 4 -- WPA/WPA2-PSK

    and two for hidden:

        * 0 -- visible
        * 1 -- hidden

    On the ESP32 port, ``scan(block=False)`` starts the scan without blocking and
    returns ``None``; the WLAN object then becomes readable (:func:`select.poll` or
    ``asyncio``) when results are ready, retrieved by `WLAN.scan_result()`.
    Readability is driven by the WLAN event register (see ``status('events')``
    below); a finished scan is the ``EVENT_SCAN`` event.  ``status('scan')`` reports
    the current scan state.  Other ports do not accept ``block``.  See the
    :ref:`ESP32 quickref <esp32_network_async>`.

    The ESP32 port also accepts keyword arguments that narrow the scan.  A directed
    scan is dramatically faster: restricting a dual-band scan to a single channel
    drops it from seconds to a few tens of milliseconds, which makes it usable inside
    a roaming loop (e.g. to measure the neighbours reported by ``status('neighbors')``).

        * ``channel`` -- scan only this channel (``0``, the default, scans all).  This
          is the main speed lever.
        * ``ssid`` -- only report access points with this SSID.  On its own it still
          scans every channel; combine it with ``channel`` to also scan quickly.
        * ``bssid`` -- only report the access point with this MAC (6 bytes).  Useful to
          measure one known AP's RSSI without connecting.
        * ``passive`` -- listen for beacons instead of sending probe requests.  Slower,
          but does not announce the station.  The driver already scans this way by
          itself on the channels where it must (5 GHz DFS).
        * ``dwell_ms`` -- time spent per channel, in milliseconds.  An ``int`` sets the
          maximum active (and passive) time; a ``(min, max)`` tuple sets the active
          scan window (see 802.11 MinChannelTime / MaxChannelTime).  ``0`` (the
          default) keeps the driver's own timing.  A short dwell trades completeness
          for speed, and it also overrides the longer listening time the driver
          reserves for passively scanned channels, so weak and 5 GHz DFS access points
          are the first to go missing.

    These keyword arguments apply when a scan is started; they are ignored on a
    ``scan(block=False)`` call that only collects an already-running scan.

.. method:: WLAN.scan_result()

    (ESP32 only.)  Retrieve the results of a non-blocking ``scan(block=False)``
    without blocking.  Returns the list of access-point tuples (in the same format
    as `WLAN.scan()`, possibly empty) once the scan has finished, or ``None`` while
    a scan is still running or none has completed.  Reading the results clears the
    ``EVENT_SCAN`` event.  Raises ``OSError`` if the scan failed.

    Blocking retrieval is done by `WLAN.scan()` itself; ``scan_result()`` never
    blocks, so it is the counterpart used with :func:`select.poll` / ``asyncio``.

.. method:: WLAN.status([param])

    Return the current status of the wireless connection.

    When called with no argument the return value describes the network link status.
    The possible statuses are defined as constants in the :mod:`network` module:

        * ``STAT_IDLE`` -- no connection and no activity,
        * ``STAT_CONNECTING`` -- connecting in progress,
        * ``STAT_WRONG_PASSWORD`` -- failed due to incorrect password,
        * ``STAT_NO_AP_FOUND`` -- failed because no access point replied,
        * ``STAT_CONNECT_FAIL`` -- failed due to other problems,
        * ``STAT_GOT_IP`` -- connection successful.

    When called with one argument *param* should be a string naming the status
    parameter to retrieve, and different parameters are supported depending on the
    mode the WiFi is in.

    In STA mode, passing ``'rssi'`` returns a signal strength indicator value, whose
    format varies depending on the port (this is available on all ports that support
    WiFi network interfaces, except for CC3200).

    In AP mode, passing ``'stations'`` returns a list of connected WiFi stations
    (this is available on all ports that support WiFi network interfaces, except for
    CC3200).  The format of the station information entries varies across ports,
    providing either the raw BSSID of the connected station, the IP address of the
    connected station, or both.

    On ESP32, in STA mode, passing ``'bssid'`` returns the MAC address (6 bytes) of
    the access point the station is currently associated with, and ``'channel'``
    returns its primary channel (this is the associated AP's channel, unlike
    ``config('channel')`` which reports the radio's momentary channel and hops during
    scans).

    On ESP32, in STA mode, passing ``'reason'`` returns the reason code of the most
    recent disconnect (one of the ``network.WLAN`` ``REASON_*`` / ESP-IDF
    ``WIFI_REASON_*`` values, ``0`` if none).  Read while disconnected it distinguishes,
    for example, an access-point-initiated deauthentication (band steering or a
    minimum-RSSI kick) from a beacon timeout (the station losing the signal).  It is
    reset to ``0`` once a connection succeeds.  Reading it acknowledges the
    ``EVENT_DISCONNECTED`` event (like ``isconnected()``).

    On ESP32, passing ``'events'`` returns a bitmask of pending asynchronous events
    (the ``WLAN.EVENT_*`` constants).  This is a pure read: the driver sets a bit when
    the event occurs and it is cleared when you read the corresponding value --
    ``scan_result()`` clears ``EVENT_SCAN``, ``isconnected()`` clears ``EVENT_CONNECTED``
    and ``EVENT_DISCONNECTED``, and ``status('stations')`` clears ``EVENT_STATIONS``.
    The WLAN object is readable (:func:`select.poll` / ``asyncio``) while any event
    enabled in the ``event_mask`` config option is pending, so a single poll can wait
    for scan completion, a connect or disconnect, or a soft-AP client change; read
    ``'events'`` to see which occurred.

    On ESP32, passing ``'scan'`` returns the current scan state as one of
    ``WLAN.SCAN_IDLE`` (no scan in progress and no result waiting), ``WLAN.SCAN_RUNNING``
    (a scan is running) or ``WLAN.SCAN_DONE`` (finished, results waiting for
    `WLAN.scan_result()`).  This is a pure read and does not clear any event.

.. method:: WLAN.isconnected()

    In case of STA mode, returns ``True`` if connected to a WiFi access
    point and has a valid IP address.  In AP mode returns ``True`` when a
    station is connected. Returns ``False`` otherwise.

.. method:: WLAN.ifconfig([(ip, subnet, gateway, dns)])

   Get/set IP-level network interface parameters: IP address, subnet mask,
   gateway and DNS server. When called with no arguments, this method returns
   a 4-tuple with the above information. To set the above values, pass a
   4-tuple with the required information.  For example::

    nic.ifconfig(('192.168.0.4', '255.255.255.0', '192.168.0.1', '8.8.8.8'))

.. method:: WLAN.config('param')
            WLAN.config(param=value, ...)

   Get or set general network interface parameters. These methods allow to work
   with additional parameters beyond standard IP configuration (as dealt with by
   `AbstractNIC.ipconfig()`). These include network-specific and hardware-specific
   parameters. For setting parameters, keyword argument syntax should be used,
   multiple parameters can be set at once. For querying, parameters name should
   be quoted as a string, and only one parameter can be queries at time::

    # Set WiFi access point name (formally known as SSID) and WiFi channel
    ap.config(ssid='My AP', channel=11)
    # Query params one by one
    print(ap.config('ssid'))
    print(ap.config('channel'))

   Following are commonly supported parameters (availability of a specific parameter
   depends on network technology type, driver, and :term:`MicroPython port`).

   =============  ===========
   Parameter      Description
   =============  ===========
   mac            MAC address (bytes)
   ssid           WiFi access point name (string)
   channel        WiFi channel (integer). Depending on the port this may only be supported on the AP interface.
   hidden         Whether SSID is hidden (boolean)
   security       Security protocol supported (enumeration, see module constants)
   key            Access key (string)
   hostname       The hostname that will be sent to DHCP (STA interfaces) and mDNS (if supported, both STA and AP). (Deprecated, use :func:`network.hostname` instead)
   reconnects     Number of reconnect attempts to make (integer, 0=none, -1=unlimited)
   txpower        Maximum transmit power in dBm (integer or float)
   pm             WiFi Power Management setting (see below for allowed values)
   protocol       (ESP32 Only.) WiFi Low level 802.11 protocol. See `WLAN.PROTOCOL_DEFAULT`.
   bandwidth      (ESP32 Only.) WiFi channel bandwidth. See `WLAN.BANDWIDTH_20` and others.
   scan_method    (ESP32 Only.) How the STA scans for the target AP when connecting. See `WLAN.SCAN_FAST` and `WLAN.SCAN_ALL_CHANNEL`.
   sort_method    (ESP32 Only.) How matching APs are ranked when connecting. See `WLAN.SORT_BY_SIGNAL` and `WLAN.SORT_BY_SECURITY`.
   rssi_threshold (ESP32 Only.) Minimum RSSI (dBm, negative integer) an AP must have to be considered when connecting. ``0`` disables the threshold. Like the other connection policy parameters it is remembered by the Wi-Fi driver, so it keeps filtering later ``connect()`` calls until it is changed back.
   failure_retry_cnt (ESP32 Only.) Number of connection retries on one AP before moving to the next candidate. Requires ``scan_method=WLAN.SCAN_ALL_CHANNEL``.
   event_mask     (ESP32 Only.) Bitmask of ``WLAN.EVENT_*`` events allowed to make the interface readable for :func:`select.poll` (default: all). See ``status('events')``.
   =============  ===========

   .. note::

      On ESP32, ``connect()`` defaults to a *fast scan* that joins the **first**
      matching AP found, which in a network with several APs of the same SSID may
      not be the strongest one. To make ``connect()`` pick the AP with the best
      signal, set an all-channel scan sorted by signal **before** calling
      ``connect()``::

         sta.config(scan_method=network.WLAN.SCAN_ALL_CHANNEL,
                    sort_method=network.WLAN.SORT_BY_SIGNAL)
         sta.connect(ssid, key)

      These settings persist across ``connect()`` calls until changed.

   .. note::

      **Wi-Fi roaming (ESP32).** Roaming lets the station move to a stronger access
      point of the same network. It is configured entirely at build time and needs
      no application code. It is off by default; a board enables it by adding the
      ``boards/sdkconfig.roaming`` fragment to its ``SDKCONFIG_DEFAULTS`` in
      ``mpconfigboard.cmake``:

      * ``boards/sdkconfig.roaming`` turns on 802.11k/v (and 802.11r fast
        transition).  The station advertises these capabilities on every
        ``connect()`` so the access point can steer it to a better BSS without
        dropping the connection (network-assisted roaming).  This requires the
        network infrastructure to actually send the steering requests.
      * ``boards/sdkconfig.roaming_app`` (add in addition to the above) turns on
        Espressif's experimental roaming app: the station also roams autonomously,
        scanning and moving to a stronger AP on low signal, entirely in the Wi-Fi
        task.  This app has no runtime switch.

      On a roaming build the station is free to move between the access points of
      the network, so pinning it to one AP with the *bssid* argument of ``connect()``
      is not honoured across roams.  Poll ``status('bssid')`` to observe when a roam
      occurred, and enable the ESP-IDF log (see `esp32.osdebug`) to see the roaming
      app's decisions.

Event constants (ESP32 only)
----------------------------

These identify the bits returned by ``status('events')`` and accepted by the
``event_mask`` config option.  The interface becomes readable for
:func:`select.poll` while any enabled event is pending; reading the associated
value clears its bit.

.. data:: WLAN.EVENT_SCAN
          WLAN.EVENT_CONNECTED
          WLAN.EVENT_DISCONNECTED
          WLAN.EVENT_STATIONS

    In order: a non-blocking ``scan()`` finished; the station obtained an IP; the
    station disconnected; a soft-AP client joined or left.

.. data:: WLAN.SCAN_IDLE
          WLAN.SCAN_RUNNING
          WLAN.SCAN_DONE

    (ESP32 only.)  The scan state returned by ``status('scan')``: no scan in
    progress and no result waiting; a scan is running; a scan has finished and its
    results are waiting to be read by `WLAN.scan_result()`.

CSI Methods (ESP32 only)
------------------------

.. note::
   These methods are only available on ESP32 builds with CSI support enabled.
   The standard generic ESP32, ESP32-C3, ESP32-C5, ESP32-C6, and ESP32-S3
   board definitions enable this in their default configuration. Other builds
   need ``CONFIG_ESP_WIFI_CSI_ENABLED=y`` in the ESP-IDF configuration.

Channel State Information (CSI) provides per-packet physical layer channel data
derived from received Wi-Fi frames. CSI capture requires an active Wi-Fi
connection and incoming traffic to the device. Without traffic, no CSI frames
will be captured.

Other Espressif CSI options are hard-coded to defaults intended for connected
station capture.

.. method:: WLAN.csi_enable(buffer_size=16)

   Enable CSI capture and allocate a circular buffer for received frames.

   The optional ``buffer_size`` argument sets the number of frames stored before
   new incoming frames are dropped. Larger values reduce drops at the cost of RAM. The
   exact maximum depends on the build, but it is limited by the underlying
   ringbuffer implementation to roughly 100 frames.

   Raises ``OSError`` if CSI cannot be enabled, for example if Wi-Fi is not
   active or the ESP-IDF rejects the configuration.

   Example::

      import network
      import time

      wlan = network.WLAN(network.WLAN.IF_STA)
      wlan.active(True)
      wlan.config(protocol=network.MODE_11B | network.MODE_11G | network.MODE_11N)
      wlan.config(pm=wlan.PM_NONE)
      wlan.connect("SSID", "password")

      while not wlan.isconnected():
          time.sleep_ms(100)

      wlan.csi_enable(buffer_size=32)

.. method:: WLAN.csi_disable()

   Disable CSI capture and clean up resources.

.. method:: WLAN.csi_read([result])

   Read a CSI frame from the buffer.

   **Returns:** A list containing CSI frame data, or ``None`` if no frames are
   available.

   If the optional ``result`` argument is provided, it must be a previous list
   returned by `WLAN.csi_read()`. The list will be updated in place and
   returned again. This reduces heap churn in busy read loops by reusing the
   existing list object and, when the captured frame fits, the existing CSI
   data ``bytearray``.

   **Frame list fields (in order):**

   * **0 - rssi** (int): Received signal strength in dBm
   * **1 - channel** (int): Wi-Fi channel number
   * **2 - mac** (bytes): Source MAC address (6 bytes)
   * **3 - timestamp** (int): Timestamp in microseconds
   * **4 - local_timestamp** (int): Local timestamp from Wi-Fi hardware
   * **5 - data** (bytearray): CSI raw data (I/Q components as int8_t values)
   * **6 - rate** (int): Data rate
   * **7 - sig_mode** (int): Signal mode (legacy, HT, VHT)
   * **8 - mcs** (int): Modulation and Coding Scheme index
   * **9 - cwb** (int): Channel bandwidth
   * **10 - smoothing** (int): Smoothing applied
   * **11 - not_sounding** (int): Not sounding frame
   * **12 - aggregation** (int): Aggregation
   * **13 - stbc** (int): STBC
   * **14 - fec_coding** (int): FEC coding
   * **15 - sgi** (int): Short GI
   * **16 - noise_floor** (int): Background noise level in dBm
   * **17 - ampdu_cnt** (int): AMPDU count
   * **18 - secondary_channel** (int): Secondary channel
   * **19 - ant** (int): Antenna
   * **20 - sig_len** (int): Signal length
   * **21 - rx_state** (int): RX state

   Some metadata fields may be ``0`` on targets where ESP-IDF does not provide
   the corresponding value in the public CSI receive structure.

.. method:: WLAN.csi_available()

   Get the number of CSI frames available in the buffer.

.. method:: WLAN.csi_dropped()

   Get the number of CSI frames dropped due to buffer overflow.
   Frames are dropped when the buffer is full and new frames arrive faster than
   they can be read. Increase ``buffer_size`` in ``csi_enable()`` to reduce
   drops.

Constants
---------

.. data:: WLAN.PM_PERFORMANCE
        WLAN.PM_POWERSAVE
        WLAN.PM_NONE

    Allowed values for the ``WLAN.config(pm=...)`` network interface parameter:

        * ``PM_PERFORMANCE``: enable WiFi power management to balance power
          savings and WiFi performance
        * ``PM_POWERSAVE``: enable WiFi power management with additional power
          savings and reduced WiFi performance
        * ``PM_NONE``: disable wifi power management


ESP32 Protocol Constants
------------------------

The following ESP32-only constants relate to the ``WLAN.config(protocol=...)``
network interface parameter:

.. data:: WLAN.PROTOCOL_DEFAULT

      A bitmap representing all of the default 802.11 Wi-Fi modes supported by
      the chip. Consult `ESP-IDF Wi-Fi Protocols`_ documentation for details.

.. data:: WLAN.PROTOCOL_LR

      This value corresponds to the `Espressif proprietary "long-range" mode`_,
      which is not compatible with standard Wi-Fi devices. By setting this
      protocol it's possible for an ESP32 STA in long-range mode to connect to
      an ESP32 AP in long-range mode, or to use `ESP-NOW long range modes
      <espnow-long-range>`.

      This mode can be bitwise ORed with some standard 802.11 protocol bits
      (including `WLAN.PROTOCOL_DEFAULT`) in order to support a mix of standard
      Wi-Fi modes as well as LR mode, consult the `Espressif long-range
      documentation`_ for more details.

      Long range mode is not supported on ESP32-C2.

.. data:: WLAN.BANDWIDTH_20
        WLAN.BANDWIDTH_40
        WLAN.BANDWIDTH_80
        WLAN.BANDWIDTH_160
        WLAN.BANDWIDTH_80_80

      Allowed values for the ``WLAN.config(bandwidth=...)`` network interface parameter:

      * ``BANDWIDTH_20``: specifies a 20MHz wide WiFi channel when in STA and AP mode
      * ``BANDWIDTH_40``: specifies a 40MHz wide WiFi channel when in STA and AP mode
      * ``BANDWIDTH_80``: specifies a 80MHz wide WiFi channel when in AP mode, may not
        be available on all ESP32 models
      * ``BANDWIDTH_160``: specifies a 160MHz wide WiFi channel when in AP mode, may not
        be available on all ESP32 models
      * ``BANDWIDTH_80_80``: specifies a multi-antenna 80MHz + 80MHz wide WiFi channel
        setup when in AP mode, may not be available on all ESP32 models.

      When in STA mode, bandwidth can only be changed when the adapter is not connected to a
      network.  In AP mode it can be changed at any time.

.. data:: WLAN.SCAN_FAST
        WLAN.SCAN_ALL_CHANNEL

      (ESP32 only.) Allowed values for the ``WLAN.config(scan_method=...)`` parameter,
      controlling how the station scans for the target AP when connecting:

      * ``SCAN_FAST``: stop scanning as soon as a matching AP is found (the default;
        fastest connect, but joins the first match, not necessarily the strongest).
      * ``SCAN_ALL_CHANNEL``: scan every channel before connecting, so the AP can be
        chosen according to ``sort_method`` (use this to join the strongest AP).

.. data:: WLAN.SORT_BY_SIGNAL
        WLAN.SORT_BY_SECURITY

      (ESP32 only.) Allowed values for the ``WLAN.config(sort_method=...)`` parameter,
      controlling how matching APs are ranked (only relevant with ``SCAN_ALL_CHANNEL``):

      * ``SORT_BY_SIGNAL``: connect to the matching AP with the strongest signal.
      * ``SORT_BY_SECURITY``: connect to the matching AP with the strongest security.

.. data:: WLAN.REASON_UNSPECIFIED
        WLAN.REASON_AUTH_EXPIRE
        WLAN.REASON_AUTH_LEAVE
        WLAN.REASON_ASSOC_TOOMANY
        WLAN.REASON_ASSOC_LEAVE
        WLAN.REASON_BEACON_TIMEOUT
        WLAN.REASON_NO_AP_FOUND
        WLAN.REASON_AUTH_FAIL
        WLAN.REASON_ASSOC_FAIL
        WLAN.REASON_HANDSHAKE_TIMEOUT
        WLAN.REASON_CONNECTION_FAIL
        WLAN.REASON_ROAMING

      (ESP32 only.) Common disconnect reason codes returned by ``status('reason')``.
      The standard 802.11 codes (``REASON_AUTH_LEAVE``, ``REASON_ASSOC_TOOMANY``,
      ``REASON_ASSOC_LEAVE`` and others) are sent by the access point, e.g. a
      band-steering or minimum-RSSI kick.  The ESP-IDF specific codes describe what
      the station saw: ``REASON_BEACON_TIMEOUT`` (lost the signal), ``REASON_ROAMING``
      (the station roamed to another AP), ``REASON_AUTH_FAIL`` / ``REASON_ASSOC_FAIL``
      (the association attempt was rejected).  These mirror the ESP-IDF
      ``WIFI_REASON_*`` values; codes not listed here can be compared numerically.

.. _ESP-IDF Wi-Fi Protocols: https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/wifi.html#wi-fi-protocol-mode
.. _Espressif proprietary "long-range" mode:
.. _Espressif long-range documentation: https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/wifi.html#long-range-lr
