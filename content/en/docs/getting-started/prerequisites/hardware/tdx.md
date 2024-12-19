---
title: Intel® Trust Domain Extensions (TDX) Host Setup
description: Host configurations for Intel TDX machines on Ubuntu
weight: 10
categories:
  - prerequisites
tags:
  - TDX
  - Canonical
  - Ubuntu
---

Intel® TDX is a Confidential Computing technology which deploys hardware-isolated,
Virtual Machines (VMs) called Trust Domains (TDs). It protects TDs from a broad range of software attacks by
isolating them from the Virtual-Machine Manager (VMM), hypervisor, and other non-TD software on the host platform.
As a result, Intel TDX enhances a platform user’s control of data security and IP protection. Also, it enhances the
Cloud Service Providers’ (CSP) ability to provide managed cloud services without exposing tenant data to adversaries.
For more information, see the [Intel TDX overview](https://www.intel.com/content/www/us/en/developer/tools/trust-domain-extensions/overview.html).

This documentation will use Ubuntu Noble 24.04 LTS (or newer) for base host OS and guest OS.

> **Note:** Other linux distributions will be added in the future.

{{% alert title="Note" color="primary" %}}
This documentation is based on publicly accessible Canonical documentation which can be found [here](https://github.com/canonical/tdx/).
{{% /alert %}}

{{% alert title="Note" color="primary" %}}
Official TDX documentation which contains in-depth specification and instruction for other OS distributions can be found [here](https://cc-enabling.trustedservices.intel.com/).
{{% /alert %}}

## 1. Setup Host OS
In this section, you will install a generic Ubuntu 24.04 server image, install necessary packages to turn
the host OS into an Intel TDX-enabled host OS and enable Intel TDX settings in the BIOS.

### 1.1 Install Ubuntu Server Image

Download and install appropriate Ubuntu Server on the host machine: [Ubuntu 24.04 server](https://releases.ubuntu.com/24.04/).

### 1.2 Enable Intel TDX in Host OS

1. Download this repository by downloading an asset file from the [releases page on GitHub](https://github.com/canonical/tdx/releases) or by cloning the repository.

   ```bash
   git clone -b noble-24.04 https://github.com/canonical/tdx.git
   cd tdx/
   ```

2. For attestation to work, you need _Production_ hardware. Run the `check-production.sh` script to verify.

    ```bash 
    sudo ./attestation/check-production.sh
    ```   

   Expected output:
    ```console
    CPU: 4th Gen Intel® Xeon® Scalable Processors (codename: Sapphire Rapids)
    Production
    ```
   
3. After confirmation run below commands: 

    ```bash
   sed -i 's/^TDX_SETUP_ATTESTATION=0/TDX_SETUP_ATTESTATION=1/' ./setup-tdx-config
   ./setup-tdx-host.sh
   ```

   > **Note:** If you're behind a proxy, use `sudo -E` to preserve user environment.

4. Configure the PCCS service:

   ```bash
   sudo /usr/bin/pccs-configure
   ```

   An example configuration:

   ```console
   Checking nodejs version ...
   nodejs is installed, continue...
   Checking cracklib-runtime ...
   Set HTTPS listening port [8081] (1024-65535) :
   Set the PCCS service to accept local connections only? [Y] (Y/N) :
   Set your Intel PCS API key (Press ENTER to skip) : <Enter your Intel PCS subscription key here>
   Choose caching fill method : [LAZY] (LAZY/OFFLINE/REQ) :
   Set PCCS server administrator password: <pccs-admin-password>
   Re-enter administrator password: <pccs-admin-password>
   Set PCCS server user password: <pccs-server-user-password>
   Re-enter user password: <pccs-server-user-password>
   Do you want to generate insecure HTTPS key and cert for PCCS service? [Y] (Y/N) :N
   ```

   > **Note 1:** The resulting config file is located at `/opt/intel/sgx-dcap-pccs/config/default.json`.

    {{% alert title="Behind proxy" color="primary" %}}
   If you're behind a proxy, run the following commands:

    sed -i 's;"proxy" :.*;"proxy" : "'${http_proxy}'",;g' /opt/intel/sgx-dcap-pccs/config/default.json
    echo "proxy type = manual" | sudo tee -a /etc/mpa_registration.conf > /dev/null
    echo "proxy url = ${http_proxy}" | sudo tee -a /etc/mpa_registration.conf > /dev/null

    {{% /alert %}}
    
5. Restart the PCCS service:

   ```bash
   sudo systemctl restart pccs
   ```

6. Reboot and enter BIOS.

### 1.3 Enable Intel TDX in the Host's BIOS

> **Note:** The following is a sample BIOS configuration.
The necessary BIOS settings or the menus might differ based on the platform that is used.
Please reach out to your OEM/ODM or independent BIOS vendor for instructions dedicated for your BIOS.

1. Go to `Socket Configuration > Processor Configuration > TME, TME-MT, TDX`.

    * Set `Memory Encryption (TME)` to `Enable`
    * Set `Total Memory Encryption Bypass` to `Enable` (Optional setting for best host OS and regular VM performance.)
    * Set `Total Memory Encryption Multi-Tenant (TME-MT)` to `Enable`
    * Set `TME-MT memory integrity` to `Disable`
    * Set `Trust Domain Extension (TDX)` to `Enable`
    * Set `TDX Secure Arbitration Mode Loader (SEAM Loader)` to `Enable`. (NOTE: This allows loading Intel TDX Loader and Intel TDX Module from the ESP or BIOS.)
    * Set `TME-MT/TDX key split` to a non-zero value

2. Go to `Socket Configuration > Processor Configuration > Software Guard Extension (SGX)`.

    * Set `SW Guard Extensions (SGX)` to `Enable`
    * Set `SGX Factory Reset` to `Enable`
    * Set `SGX Auto MP Registration` to `Enable`

3. Save the BIOS settings and boot up.

### 1.4 Verify Intel TDX is Enabled on Host OS

1. Verify that Intel TDX is enabled using the `dmesg` command:

    ```bash
    sudo dmesg | grep -i tdx
    ```

    The message `virt/tdx: module initialized` proves that Intel TDX has initialized properly. Here is an example output:
    
    ```console
    ...
    [    5.205693] virt/tdx: BIOS enabled: private KeyID range [64, 128)
    [   29.884504] virt/tdx: 1050644 KB allocated for PAMT
    [   29.884513] virt/tdx: module initialized
    ...
    ```

2. Check the log of the MPA service:

   ```bash
   sudo systemctl status mpa_registration_tool
   ```

   Example output:

   ```console
   mpa_registration_tool.service - Intel MPA Registration
       Loaded: loaded (/usr/lib/systemd/system/mpa_registration_tool.service; enabled; preset: enabled)
       Active: inactive (dead) since Tue 2024-04-09 22:54:50 UTC; 11h ago
   Duration: 46ms
   Main PID: 3409 (code=exited, status=0/SUCCESS)
            CPU: 21ms

   Apr 09 22:54:50 right-glider-515046 systemd[1]: Started mpa_registration_tool.service - Intel MPA Registratio>
   Apr 09 22:54:50 right-glider-515046 systemd[1]: mpa_registration_tool.service: Deactivated successfully.
    ``` 

3. Check the log file of the MPA:

   ```bash 
   cat /var/log/mpa_registration.log 
   ``` 

   An example output of successful registration:

   ```console
   [04-06-2024 03:05:53] INFO: SGX Registration Agent version: 1.20.100.2
   [04-06-2024 03:05:53] INFO: Starts Registration Agent Flow.
   [04-06-2024 03:05:54] INFO: Registration Flow - PLATFORM_ESTABLISHMENT or TCB_RECOVERY passed successfully.
   [04-06-2024 03:05:54] INFO: Finished Registration Agent Flow.
   ```


## 2. Troubleshooting Tips

### 2.1 Performance is poor

Ensure you're using the latest TDX module. 
You can check the current version with `dmesg` (the version line looks like: `virt/tdx: TDX module: attributes 0x0, vendor_id 0x8086, major_version 1, minor_version 5, build_date 20240129, build_num 698`). 
See [link](https://cc-enabling.trustedservices.intel.com/intel-tdx-enabling-guide/04/hardware_setup/#deploy-specific-intel-tdx-module-version) on ways to update your TDX module. 

> **Note:** If you chose to "Update Intel TDX Module via Binary Deployment", make sure you're using the correct TDX module version for your hardware.

### 2.2 TDX is not enabled on the host

1. Ensure your installation of the TDX host components using `setup-tdx-host.sh` did not have any errors. 
2. Ensure BIOS settings are correct.

### 2.3 Installation seems to hang

1. Verify you can get out to the Internet.
2. If you're behind a proxy, make sure you have proper proxy settings. 
3. If you're behind a proxy, use `sudo -E` to preserve user environment.

### 2.4 MPA Registration failed

1. Check log of `/var/log/mpa_registration.log`.

    If it contains error like below:

    ``` console
    [05-12-2024 06:31:16] INFO: SGX Registration Agent version: 1.12.100.3
    [05-12-2024 06:31:16] INFO: Starts Registration Agent Flow.
    [05-12-2024 06:43:50] INFO: SGX MP Server configuration flag indicates that Registration Server won't save encrypted platform keys.
    [05-12-2024 06:43:50] INFO: Platform registration request (PLATFORM_MANIFEST) won't be send to Registration Server.
    [05-12-2024 06:43:51] INFO: Please use managment tool or PCKCertIDRetrivalTool to read PLATFORM_MANIFEST.
    [05-12-2024 06:43:51] INFO: Finished Registration Agent Flow.
    ```
   
2. Re-do the registration from scratch with these steps:

   1. Remove the PCCS cache file:  `sudo rm /opt/intel/sgx-dcap-pccs/pckcache.db`.
   2. Remove the MPA log file:  `sudo rm /var/log/mpa_registration.log`.
   3. Reboot.
   4. Go into the BIOS.
   5. Navigate to `Socket Configuration > Processor Configuration > Software Guard Extension (SGX)`.
   6. Set these in this specific order:
       - `SGX Factory Reset` to `Enable`
       - `SGX Auto MP Registration` to `Enable`
   7. Save the BIOS settings and boot up.