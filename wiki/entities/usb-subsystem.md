---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [professional-linux-kernel-architecture]
tags: [usb, device-drivers, urb, host-controller, hardware]
---

# USB Subsystem

## Overview

The Linux USB subsystem implements the host side of the Universal Serial Bus specification. It provides a layered architecture with three tiers: host controller drivers (HCD) at the bottom, the USB core in the middle, and USB device drivers at the top. The subsystem handles device enumeration, configuration, and data transfer across all USB speed classes supported in the 2.6.24 kernel (Low, Full, High, and Super speed).

USB device drivers never interact with the hardware directly. Instead, they submit I/O requests through the USB core, which routes them to the appropriate host controller driver. This layered design means a single USB device driver works with any host controller.

## USB Device Model

The USB subsystem represents hardware through a hierarchy of kernel structures that mirrors the physical USB topology: devices contain configurations, configurations contain interfaces, and interfaces contain endpoints.

### struct usb_device

Represents a physical USB device attached to the bus. Key fields include:

- `devnum` -- the device address assigned on the bus (1--127)
- `speed` -- connection speed: `USB_SPEED_LOW`, `USB_SPEED_FULL`, `USB_SPEED_HIGH`, or `USB_SPEED_SUPER`
- `state` -- lifecycle state, ranging from `USB_STATE_NOTATTACHED` through `USB_STATE_POWERED`, `USB_STATE_ADDRESS`, `USB_STATE_DEFAULT`, and finally `USB_STATE_CONFIGURED`
- `config` -- array of all available configurations read from the device
- `actconfig` -- pointer to the currently active configuration
- `ep0` -- the default control endpoint (endpoint 0), used for device setup and management
- `manufacturer`, `product`, `serial` -- string descriptors read from the device
- `descriptor` -- the USB device descriptor containing vendor ID, product ID, device class, and protocol information

### struct usb_interface

Represents one interface within a configuration. This is the level at which drivers bind -- a single physical device may expose multiple interfaces, each driven by a different driver (for example, a USB headset might have separate audio and HID interfaces). Key fields:

- `cur_altsetting` -- pointer to the currently selected alternate setting
- `altsetting` -- array of all available alternate interface descriptors
- `num_altsetting` -- number of alternate settings available

### struct usb_host_endpoint

Represents a single endpoint, which is a unidirectional data pipe on the device. Key fields:

- `desc` -- the endpoint descriptor containing the endpoint address, direction (IN or OUT), transfer type (control, bulk, interrupt, or isochronous), and `wMaxPacketSize`
- `urb_list` -- queue of pending [URBs](usb-subsystem.md#usb-request-blocks-urbs)|URBs.md) waiting to be processed for this endpoint

### Device Hierarchy

The USB device model follows a strict containment hierarchy:

```
usb_device
 └── configuration (one active at a time)
      └── usb_interface (one or more per configuration)
           └── usb_host_endpoint (one or more per interface)
```

A device always has endpoint 0 (the default control endpoint) available regardless of which configuration or interface is active.

## USB Request Blocks (URBs)

### struct urb

The URB is the fundamental I/O structure for all USB data transfers. Every communication between a driver and a USB device passes through a URB. Key fields:

- `dev` -- pointer to the target `usb_device`
- `pipe` -- encoded value combining endpoint address, direction, and transfer type (constructed with `usb_sndbulkpipe()`, `usb_rcvintpipe()`, and similar macros)
- `transfer_buffer` / `transfer_buffer_length` -- the data buffer and its size
- `actual_length` -- number of bytes actually transferred (set by the HCD on completion)
- `status` -- completion status (0 for success, negative errno on failure)
- `complete` -- callback function invoked when the URB finishes
- `context` -- opaque pointer passed to the completion callback, typically a driver-private structure

### Transfer Types

USB defines four transfer types, each suited to different data patterns:

- **Control** -- used for device setup and management. Consists of a setup phase, optional data phase, and status phase. Every device must support control transfers on endpoint 0.
- **Bulk** -- reliable transfers of large data volumes with no timing guarantees. Uses all available bandwidth not reserved by other transfer types. Common for storage devices and network adapters.
- **Interrupt** -- small, periodic transfers with guaranteed latency. The host controller polls the device at a driver-specified interval. Used by keyboards, mice, and other HID devices.
- **Isochronous** -- periodic transfers with guaranteed bandwidth but no error retry. Used for audio and video streaming where timely delivery matters more than perfect accuracy.

### URB Lifecycle

A URB follows a well-defined lifecycle:

1. **Allocate** -- `usb_alloc_urb(iso_packets, gfp_flags)` allocates and initializes a URB
2. **Fill** -- helper functions populate the URB fields: `usb_fill_control_urb()`, `usb_fill_bulk_urb()`, `usb_fill_int_urb()` (isochronous URBs are filled manually)
3. **Submit** -- `usb_submit_urb(urb, gfp_flags)` queues the URB for processing
4. **Completion** -- the HCD invokes the `complete` callback when the transfer finishes (or fails)
5. **Resubmit or free** -- the driver either resubmits the URB (common for interrupt endpoints) or frees it with `usb_free_urb()`

URBs are asynchronous by design: `usb_submit_urb()` returns immediately after queueing the request. The completion callback fires later from interrupt context. Drivers that need synchronous behavior can use the convenience wrappers `usb_control_msg()` and `usb_bulk_msg()`, which block until the transfer completes.

## USB Driver Structure

### struct usb_driver

Defines a USB device driver. Key fields:

- `name` -- human-readable driver name
- `probe` -- called by the USB core when a matching device interface is found; the driver should verify the hardware and initialize its private state
- `disconnect` -- called when the device is removed or the driver is unloaded; must release all resources and cancel outstanding URBs
- `id_table` -- array of `usb_device_id` entries describing which devices this driver supports
- `suspend` / `resume` -- power management callbacks for system sleep transitions

### struct usb_device_id

Specifies matching criteria used by the USB core to bind drivers to device interfaces:

- `match_flags` -- bitmask indicating which fields to compare (`USB_DEVICE_ID_MATCH_VENDOR`, `USB_DEVICE_ID_MATCH_PRODUCT`, etc.)
- `idVendor` / `idProduct` -- match specific vendor and product IDs
- `bDeviceClass` / `bDeviceSubClass` / `bDeviceProtocol` -- match device class codes
- `bInterfaceClass` / `bInterfaceSubClass` / `bInterfaceProtocol` -- match interface class codes

The `USB_DEVICE()` and `USB_DEVICE_INFO()` macros simplify the construction of common match patterns.

### Registration and Matching

Drivers register with the USB core using `usb_register()` and unregister with `usb_deregister()`. When a new device is enumerated, the USB core iterates over each interface in the active configuration and compares it against the `id_table` of every registered driver. When a match is found, the core invokes the driver's `probe` function.

## Host Controller Drivers

### struct usb_hcd

Represents an instance of a USB host controller. The HCD structure contains the controller's root hub (itself a `usb_device`), shared hardware state, and a pointer to the `hc_driver` operations table.

### struct hc_driver

Defines the operations that a host controller must implement:

- `urb_enqueue` -- submit a URB for scheduling on the bus
- `urb_dequeue` -- cancel a pending URB
- `endpoint_disable` -- clean up when an endpoint is no longer in use
- `hub_control` -- handle hub-class requests for the root hub
- `get_frame_number` -- return the current USB frame number

### HCD Types

The 2.6.24 kernel includes drivers for several host controller specifications:

- **UHCI** (Universal Host Controller Interface) -- Intel's USB 1.x controller design
- **OHCI** (Open Host Controller Interface) -- a competing USB 1.x standard used by most non-Intel chipsets
- **EHCI** (Enhanced Host Controller Interface) -- USB 2.0 high-speed controller, often paired with a UHCI or OHCI companion for low/full-speed devices
- **xHCI** (Extensible Host Controller Interface) -- USB 3.0 controller (initial support)

Host controller drivers schedule URBs onto the physical bus, managing bandwidth allocation, frame timing, and transaction retries according to the USB specification.

## Device Enumeration

The hub driver, which is built into the USB core, monitors hub port status change events. When a new device is detected, enumeration proceeds through a defined sequence:

1. The hub driver detects a port status change (device insertion)
2. The port is reset to put the device into the Default state
3. `usb_set_address()` assigns a unique bus address to the device
4. The device descriptor is read from endpoint 0 to determine basic device properties
5. Configuration descriptors are read to discover available interfaces and endpoints
6. A configuration is selected (typically the first one)
7. `usb_new_device()` calls `device_add()`, which triggers the driver model to match the device's interfaces against registered USB drivers
8. For each matched interface, the driver's `probe` function is called

The entire process is managed by the kernel's `khubd` thread (the hub daemon), which handles enumeration events for all USB buses in the system.

## USB Core

The USB core (`drivers/usb/core/`) sits between device drivers and host controller drivers, providing the central coordination layer.

### Key Responsibilities

- **Bus topology management** -- maintains the tree of hubs and devices reflecting the physical USB topology
- **Driver API** -- provides both asynchronous (`usb_submit_urb`) and synchronous (`usb_control_msg`, `usb_bulk_msg`) interfaces for data transfer
- **Bandwidth allocation** -- tracks periodic (interrupt and isochronous) bandwidth reservations to prevent overcommitting the bus
- **Power management** -- supports device suspend, resume, and remote wakeup
- **Device lifecycle** -- handles enumeration, configuration, and removal

### Sysfs Representation

USB devices appear in sysfs under `/sys/bus/usb/devices/`. Each device is named using its bus number and port path (e.g., `2-1.3` means bus 2, port 1, port 3 on a downstream hub). This representation allows userspace tools like `lsusb` and udev rules to identify and manage USB devices.

## See also

- [device-driver-model](device-driver-model.md)
- [kernel-modules](kernel-modules.md)
- [devices](devices.md)
