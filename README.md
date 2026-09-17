# IT Lab: Firewall, Domain Controller, Ticketing & Client Build

A home-lab environment built in VirtualBox: an OPNsense firewall/router, a Windows Server 2022 domain controller (AD DS, DNS, DHCP), a Linux ticketing server (osTicket), and a domain-joined Windows client.

![Lab network diagram](media/image43.png)

## Lab Topology

| Role | OS | Key Config |
|---|---|---|
| Firewall / Router | OPNsense | WAN: NAT (VirtualBox), LAN: `192.168.50.1/24` |
| Domain Controller | Windows Server 2022 (Desktop Experience) | Domain: `lab.local`, static IP on the LAN subnet, DNS points to itself, runs AD DS + DNS + DHCP |
| Ticketing Server | Ubuntu Server (osTicket) | DHCP lease from the DC, internal network |
| Client | Windows 10/11 Pro | Domain-joined to `lab.local` |

All VMs (other than the firewall's WAN side) sit on the same VirtualBox **Internal Network** — the name must match **exactly**, character-for-character, on every adapter, or the VMs won't be able to reach each other.

## Prerequisites

- VirtualBox
- `qemu-img` (for converting the OPNsense raw image to VMDK)
- ISOs/images:
  - [OPNsense](https://opnsense.org/download/) (raw `.img.bz2`)
  - [Windows Server 2022](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022)
  - [Ubuntu Server](https://ubuntu.com/download/server)
  - [Windows 10](https://www.microsoft.com/en-us/software-download/windows10ISO)

## 1. Firewall (OPNsense)

1. Download the OPNsense image.

   ![OPNsense download page](media/image4.png)

2. Decompress the `.bz2` archive.

   ![Uncompressing the image](media/image38.png)

3. Open a terminal and rename the extracted file for convenience.

   ![Terminal, renaming the file](media/image25.png)

4. Use `bunzip2` to extract the `.bz2` file — you should end up with this:

   ![Extracted file result](media/image42.png)

5. Make sure `qemu-img` is installed, then convert the raw image to VMDK (VirtualBox can't boot a raw image directly — it needs a container format like VMDK).

   ![qemu-img installed](media/image9.png)

   ![Conversion command](media/image58.png)

6. Delete the raw `.img` file and move the `.vmdk` out of the Downloads folder.

   ![Moving the vmdk file](media/image40.png)

7. Rename `output.vmdk` to `OPNsense.vmdk`.

   ![Renamed vmdk](media/image50.png)

8. In VirtualBox, create a new VM:
   - Leave the ISO field blank (booting from the VMDK instead).
   - OS type: **BSD** (OPNsense is FreeBSD-based, not Linux).

   ![New VM, ISO blank, BSD type](media/image17.png)

9. Size the virtual hardware to taste — the lab used modest specs to save resources.

   ![Virtual hardware settings](media/image63.png)

10. Right-click the new VM in the list and enable **Expert Mode** while configuring networking.

    ![VM list, expert mode](media/image39.png)

11. Configure two network adapters:
    - **Adapter 1 (LAN):**

      ![Adapter one settings](media/image8.png)
      ![LAN adapter](media/image68.png)

    - **Adapter 2 (WAN):**

      ![WAN adapter](media/image61.png)

12. Under **Storage**, remove the default VDI, click the attachment icon, and add the `OPNsense.vmdk`.

    ![Attaching the vmdk under storage](media/image51.png)

13. Start the VM.

    ![Starting OPNsense](media/image36.png)

    ![Booting](media/image26.png)

14. At the console, answer **No** to LAGGs (single NIC per side — nothing to bond) and **No** to VLANs (one flat subnet — nothing to tag).

    ![No for LAGGs](media/image33.png)
    ![No for VLANs](media/image45.png)

15. Note the MAC addresses shown, then match them back to the adapters configured in VirtualBox to identify which is WAN and which is LAN.

    ![Noting MAC addresses](media/image60.png)
    ![Matching MAC addresses](media/image20.png)

16. Assign the WAN interface (attached to NAT) — e.g. `em1` if its MAC ends in `394`.

    ![Assigning WAN interface](media/image70.png)

17. Add the LAN interface and skip the rest.

    ![Adding LAN interface](media/image34.png)

18. Log in at the console with the default credentials:

    **Username:** `root`
    **Password:** `opnsense`

    ![Default login screen](media/image72.png)

19. From the main menu: `2` → Set interface IP addresses → `1` for LAN. Decline DHCP on the LAN interface (the Windows DC will be the DHCP server — two DHCP servers on one subnet causes lease conflicts). Set the LAN IP to **192.168.50.1 / 24**, decline the remaining prompts back to the main menu.

    ![Setting the LAN IP](media/image23.png)

20. Leave OPNsense running (or headless, to save resources) — the DC needs it as the default gateway for internet access.

## 2. Domain Controller (Windows Server 2022)

1. Download the Server 2022 ISO, move and rename it into your images folder.

   ![Moving/renaming the ISO](media/image75.png)

2. Create a new VM named `DC_Windows` and attach the ISO.

   ![New VM, DC_Windows](media/image67.png)

3. Set credentials for the VM.

   ![Username/password step 1](media/image59.png)
   ![Username/password step 2](media/image56.png)

4. Allocate **4 GB RAM / 2 vCPUs**.

   ![4GB RAM, 2 processors](media/image73.png)

5. Allocate **20 GB** of disk space.

   ![Disk space allocation](media/image3.png)

6. Make sure the network adapter is connected to the shared **Internal Network** — same name as the firewall's LAN adapter (a mismatch isolates the DC from the firewall).

   ![Network adapter, internal network](media/image15.png)

7. Start the VM and proceed through the Windows installation.

   ![Starting the VM](media/image41.png)

8. Choose the **Desktop Experience** edition.

   ![Desktop Experience option](media/image49.png)

9. Use custom install and select the unallocated space.

   ![Custom install, unallocated space](media/image52.png)

10. Set the local administrator password.

    ![Setting the admin password](media/image18.png)

11. Check the network settings (globe icon → Network settings → Change adapter options → Ethernet → Details). You'll see an APIPA address (`169.254.x.x`) — expected, since no DHCP server exists yet.

    ![APIPA address shown](media/image69.png)

12. Set a static IP manually (Ethernet adapter → Properties → TCP/IPv4 → Properties):
    - IP address per your subnet plan
    - Default gateway: the OPNsense LAN IP (`192.168.50.1`)
    - DNS: loopback (`127.0.0.1`) — once AD DS is installed, this server hosts its own AD-integrated DNS zone.

    ![TCP/IPv4 settings](media/image57.png)

13. **Server Manager → Add Roles**: install **DHCP**, **DNS**, and **AD DS** together (DHCP will later be authorized against AD DS, and AD DS needs DNS to integrate with).

    ![Adding roles](media/image54.png)

14. Click through to install, then restart the server.

    ![Install and restart](media/image62.png)

15. Click the notification flag and choose **Promote this server to a domain controller**.

    ![Promote to domain controller](media/image5.png)

16. Create a new forest, set the domain name to `lab.local`, and click through to install.

    ![Setting the domain to lab.local](media/image64.png)

17. Set a DSRM password and click through to install — the server restarts automatically.

    ![DSRM password](media/image31.png)
    ![Restarting](media/image10.png)
    ![Restart complete](media/image24.png)

18. Back in Server Manager, go to **Tools**.

    ![Server Manager Tools menu](media/image29.png)

19. Click **DHCP**.

    ![DHCP console](media/image2.png)

20. Right-click the DHCP node under IPv4 and create a **New Scope**.

    ![Right-click DHCP, new scope](media/image13.png)
    ![New Scope wizard](media/image27.png)

21. Set the scope to match your subnet.

    ![Scope matching subnet](media/image7.png)

22. No exclusions needed.

    ![No exclusions](media/image1.png)

23. It's a lab, so lease duration doesn't matter much.

    ![Lease duration](media/image30.png)

24. Set the router/default gateway option to the OPNsense LAN IP.

    ![Default gateway option](media/image44.png)

25. Click through to finish, and confirm the scope is activated.

    ![Back to DHCP screen](media/image47.png)

26. Right-click the DHCP server node and choose **Authorize** — this is what actually turns leasing on; an unauthorized DHCP server silently refuses to hand out leases on an AD network, even with the service running and scope active.

    ![Authorizing the DHCP server](media/image19.png)

27. Refresh — the DHCP server node should turn green.

    ![DHCP server authorized, green](media/image22.png)

28. Open `services.msc` (Win+R).

    ![services.msc](media/image71.png)

29. Find the DHCP Server service and confirm it's set to **Automatic** startup, so it survives reboots without manual intervention.

    ![DHCP Server service, automatic startup](media/image74.png)

30. Create a domain user account to use on the client machine later.

    ![Creating a new user, step 1](media/image66.png)
    ![Creating a new user, step 2](media/image12.png)
    ![Setting the user's password](media/image11.png)

## 3. Ticketing Server (osTicket on Ubuntu)

1. Download [Ubuntu Server](https://ubuntu.com/download/server).

   ![Ubuntu Server download page](media/image35.png)

2. Create a VM, attach the ISO, give it a name.

   ![New VM with ISO attached](media/image55.png)

3. Allocate **4 GB RAM** (recommended to avoid lag).

   ![4GB RAM allocation](media/image6.png)

4. Make sure the network adapter is connected to the same Internal Network so it can pull a DHCP lease from the DC.

   ![Network adapter, internal](media/image14.png)

5. After install, confirm it has a DHCP lease and internet connectivity, then update the system:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```
6. Install your ticketing platform of choice (e.g. osTicket).

## 4. Windows Client

1. Download the [Windows 10 ISO](https://www.microsoft.com/en-us/software-download/windows10ISO) and keep it with the rest of your images/ISOs.

   ![Windows 10 download](media/image48.png)

2. Create the VM, attach the ISO, confirm it picks up a DHCP lease.

   ![Checking DHCP lease](media/image28.png)

3. If you end up with a **Home** edition, upgrade to **Pro** (Home can't join an Active Directory domain — only Pro/Enterprise/Education can).

   ![Home edition, needs upgrade to Pro](media/image37.png)

4. Settings → Update & Security → Activation → **Change product key**, enter Microsoft's generic Pro KMS client key and let it reboot:

   ```
   VK7JG-NPHTM-C97JM-9MPGT-3V66T
   ```

   ![Changing the product key](media/image65.png)

5. Start the upgrade. This switches the edition to Pro; it still needs a genuine license to fully activate. If it won't let you upgrade, temporarily disable the network adapter and try again.

   ![Starting the upgrade](media/image16.png)

   ![Retry with network adapter disabled](media/image46.png)

6. Go to **Settings → About → Advanced system settings → Computer Name**.

   ![Advanced system settings](media/image21.png)

7. Click **Change**.

   ![Change computer name/domain](media/image32.png)

8. Type the domain exactly as configured on the DC: `lab.local`, then authenticate with the domain user created earlier (e.g. `ali@lab.local`).

   ![Joining lab.local](media/image53.png)

## Troubleshooting Notes

- **VMs can't see each other:** almost always an Internal Network name mismatch — it's case- and character-sensitive across every adapter.
- **DC shows an APIPA (169.254.x.x) address:** normal before DHCP is configured; not an error.
- **DHCP scope active but no clients getting leases:** the DHCP server is probably unauthorized in AD — right-click it in the DHCP console and choose Authorize.
- **Client won't join the domain:** confirm the edition is Pro/Enterprise/Education and that the domain name was typed exactly as `lab.local`.
