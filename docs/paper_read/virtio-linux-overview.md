# virtio: Towards a De-Facto Standard For Virtual I/O Devices

## Summary

VirtIO is an abstraction layer over virtual I/O devices in Linux, **providing a common and efficient interface** to handle block, network, and other I/O operations with guests in a standardized way.

## Outline

* VirtIO: The Three Goal

    * Driver Unification
    * Common ABI for general publication and use of buffers
    * Complete ABI implementations

* The configuration operations

    * Reading and writing feature bits
    * Accessing the configuration space
    * Managing the status bits
    * Performing a device reset

* Host-Guest Transport

    * Abstraction: Virtqueues
    * Implementation: VirtIO_Ring

* Current VirtIO Drivers
    * VirtIO-Block
    * VirtIO-Net
