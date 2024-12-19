---
title: Secure Execution Host Setup 
description: Host configurations for IBM s390x
weight: 10
categories:
- prerequisites
tags:
- SE
---

## Platform Setup

This document outlines the steps to configure a host machine to support
IBM Secure Execution on IBM Z & LinuxONE platforms. This capability enables
enhanced security for workloads by taking advantage of protected
virtualization. Ensure the host meets the necessary hardware and software
requirements before proceeding.

### Hardware Requirements

Supported hardware includes these systems:

- IBM z15 or newer models
- IBM LinuxONE III or newer models

### Software Requirements

Additionally, the system must meet specific CPU and kernel configuration
requirements. Follow the steps below to verify and enable the Secure Execution
capability.

1. Verify Protected Virtualization Support in the Kernel

    Run the following command to ensure the kernel supports protected virtualization:
    ```bash
    cat /sys/firmware/uv/prot_virt_host
    ```
    A value of 1 indicates support.

2. Check Ultravisor Memory Reservation

    Confirm that the ultravisor has reserved memory during the current boot:
    ```bash
    sudo dmesg | grep -i ultravisor
    ```
    Example output:
    ```
    [    0.063630] prot_virt.f9efb6: Reserving 98MB as ultravisor base storage
    ```

3. Validate the Secure Execution Facility Bit

    Ensure the required facility bit (158) is present:
    ```bash
    cat /proc/cpuinfo | grep 158
    ```
    The facilities field should include 158.

If any required configuration is missing, contact your cloud provider to
enable the Secure Execution capability for a machine. Alternatively, if
you have administrative privileges and the facility bit (158) is set, you can
enable it by modifying kernel parameters and rebooting the system:

1. Modify Kernel Parameters

    Update the kernel configuration to include the prot_virt=1 parameter:
    ```bash
    sudo sed -i 's/^\(parameters.*\)/\1 prot_virt=1/g' /etc/zipl.conf
    ```

2. Update the Bootloader and reboot the System

    Apply the changes to the bootloader and reboot the system:
    ```bash
    sudo zipl -V
    sudo systemctl reboot
    ```

3. Repeat the Verification Steps

    After rebooting, repeat the verification steps above to ensure Secure Execution is properly enabled.

### Additional Notes

- The steps to enable Secure Execution might vary depending on the Linux
  distributions. Consult your distribution’s documentation if necessary.
- For more detailed information about IBM Secure Execution for Linux, see also
  the official documentation at [IBM Secure Execution for Linux](https://www.ibm.com/docs/en/linux-on-systems?topic=security-secure-execution-linux).
