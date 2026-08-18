# SAPPHIFY CAN protocol

The normative CAN protocol specification for every SAPPHIFY FRC device.

**[Read the specification](SAPPHIFY_CAN_SPECIFICATION.md)** — draft v0.9.

Any team can write its own driver, replay its own logs, or integrate a SAPPHIFY device into a
non-WPILib stack from this document alone. No SAPPHIFY software is required and no part of it is
withheld.

The incumbent does not publish an equivalent: there is no frame layout, arbitration-ID reference
or DBC anywhere in CTRE's documentation. Publishing ours is the difference between a device you
can only use and a device you can understand.

Devices defined here:

- **ROTEM** — CAN FD attitude and heading reference. Sections 3.1 and 3.2.
- Sections 2 through 3.4 are common to every device: addressing, health flags, identity,
  configuration persistence, time synchronisation and firmware update.

The document is versioned with the firmware: a released firmware image always matches a released
specification version exactly. Anything not yet decided is listed as an open gap in section 4
rather than left vague.

Licence: CC BY 4.0. Implementations of the protocol are unrestricted.
