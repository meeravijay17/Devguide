# u-blox M8P RTK GPS Configuration

RTK (Real Time Kinematic) increases GPS accuracy to centimeter-level. It uses measurements of the phase of the signal's carrier wave, rather than the information content of the signal, and relies on a single reference station to provide real-time corrections, providing up to centimetre-level accuracy.

PX4 currently only supports the single-frequency (L1) u-blox M8P based GNSS receivers for RTK.

> **Note** This page describes how RTK is integrated into PX4, if you are only
> interested to know how to use it, read the [relevant page](https://docs.px4.io/en/advanced_features/rtk-gps.html)
> on the PX4 User Guide.


Two M8P GPS modules and a datalink are required to setup RTK with PX4. The GPS unit
on the ground (fixed position) is called the Base, and the in-air unit is
called the Rover. The Base unit connects to QGroundControl (via USB) and uses
the datalink to the vehicle to stream RTCM corrections to it (using the MAVLink
`GPS_RTCM_DATA` message). On the autopilot, the MAVLink packets are unpacked and
sent to the airborne GNSS unit where they are processed to get the RTK solution.

The datalink should typically be able to handle an uplink rate of 300 bytes per
second. See the Uplink Datarate section below for more.

### Automatic Configuration

Both QGroundControl and the autopilot firmware share the same [PX4 GPS driver stack](https://github.com/PX4/GpsDrivers). In practice, this means that support for new protocols and/or messages only need to be added to one place.

The PX4 GPS stack automatically sets up the u-blox M8P modules to send and receive the correct messages over the UART or USB, depending on where the module is connected (to QGroundControl or the autopilot). No configuration using U-Center is necessary.

As soon as the autopilot receives `GPS_RTCM_DATA` mavlink messages, it
automatically forwards the RTCM data to the attached GPS module.

> **Note** u-blox has two variants of the M8P chip, the M8P-0 and the M8P-2. The
> M8P-0 can only be used as Rover, not as Base, whereas the M8P-2 can be used
> both as Rover or as Base.

#### RTCM Messages

QGroundControl configures the RTK base station to output the following RTCM3.2 frames, each with 1 Hz:
- **1005** - Station coordinates XYZ for antenna reference point (Base position).
- **1077** - Full GPS pseudo-ranges, carrier phases, Doppler and signal strength (high resolution).
- **1087** - Full GLONASS pseudo-ranges, carrier phases, Doppler and signal strength (high resolution).


### Uplink Datarate

The raw RTCM messages from the base are packed into a MAVLink `GPS_RTCM_DATA` message and sent over the datalink. The length of each MAVLink message is 182 bytes, and it encapsulates RTCM messages in its body. Depending on the RTCM message, the MAVLink message is almost never completely filled.

The Base Position message (1005) is of length 22 bytes, while the others are all of variable length depending on the number of visible satellites and the number of signals from the satellite (only 1 for L1 units like M8P). Since at a given time, the _maximum_ number of satellites visible from any single constellation is 12, under real-world conditions, an uplink rate of 300 B/s is sufficient in theory.

If **MAVLink 1** is used, no packet truncation is done. Therefore the whole 182-byte `GPS_RTCM_DATA` message is sent for every RTCM message. This means that the approximate uplink requirement is increased to 700+ bytes per second, which can lead to link saturation on low-bandwidth half-duplex telemetry modules like 3DR radios.

If **MAVLink 2** is used (PX4 automatically switches to MAVLink 2 if the GCS and telemetry modules support it), empty space in a packet is truncated, leading to a much leaner uplink requirement of ~300 bytes per second. It is important that MAVLink 2 is used  on low-bandwidth links for good RTK performance. So care must be taken to make sure that the telemetry chain uses MAVLink 2 throughout. You can verify the protocol version by using the `mavlink status` command on the system console : 

```
nsh> mavlink status
instance #0:
        GCS heartbeat:  593486 us ago
        mavlink chan: #0
        type:           3DR RADIO
        rssi:           219
        remote rssi:    219
        txbuf:          94
        noise:          61
        remote noise:   58
        rx errors:      0
        fixed:          0
        flow control:   ON
        rates:
        tx: 1.285 kB/s
        txerr: 0.000 kB/s
        rx: 0.021 kB/s
        rate mult: 0.366
        accepting commands: YES
        MAVLink version: 2
        transport protocol: serial (/dev/ttyS1 @57600)
```

