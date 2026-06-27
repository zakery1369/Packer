# Packer
Automating VMware Virtual Machine Creation with Packer

This guide demonstrates how to use HashiCorp Packer to automate the creation of virtual machines on a VMware ESXi host. Packer allows you to define your virtual machine configuration and provisioning steps in code, enabling consistent and repeatable builds.

Prerequisites:

* A VMware ESXi host.

* Network connectivity between your machine running Packer and the ESXi host.

* A base virtual machine template (VMX file and associated VMDK files) on an ESXi datastore.

* SSH access enabled on your ESXi host.

* Sudo privileges on the machine where you will run Packer commands.

## Install Packer

First, you need to install Packer on the machine you'll use to orchestrate the builds (e.g., your Linux workstation or a build server). These commands are for Debian/Ubuntu-based systems:

### Add the HashiCorp GPG key

```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

### Add the official HashiCorp Linux repository

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
```

### Update package list and install Packer

```bash
sudo apt update && sudo apt install packer
```

### Verify Packer Installation

Check that Packer was installed correctly by running:

```bash
packer version
```

### Install Required Packer Plugins

Packer uses plugins to communicate with different platforms. Install the necessary plugins for VMware vSphere (which covers ESXi direct connections) and VMware Workstation/Fusion (though vmware-vmx primarily uses the vSphere plugin for remote builds):

```bash
packer plugins install github.com/hashicorp/vsphere
packer plugins install github.com/hashicorp/vmware
```

Note: Even though we connect directly to ESXi (remote\_type: "esx5"), the vsphere plugin provides the core functionality.

## Configure ESXi Host Settings

For Packer to effectively communicate and manage the VM build process, a couple of ESXi host settings are recommended or required:

a) Enable SSH Service: Packer needs SSH access to the ESXi host to execute commands for copying files and managing the VM registration. Ensure the SSH service is running on your ESXi host. You can typically enable this via the ESXi Direct Console User Interface (DCUI) under "Troubleshooting Options" or through the vSphere Client / ESXi Host Client under `Manage -> Services -> TSM-SSH`.

b) (Optional but Recommended) Enable Net.GuestIPHack: This advanced setting helps Packer discover the guest VM's IP address, especially if VMware Tools takes time to start or report the IP after the first boot.You can typically enable this via the ESXi Direct Console User Interface (DCUI) under "Troubleshooting Options" or through the vSphere Client / ESXi Host Client under `Manage -> System -> Net.GuestIPHack`.

```bash
Net.GuestIPHack = "1"
```

### Download and Install VMware OVF Tool (Optional)

While our example configuration uses "skip\_export": true (meaning Packer won't create an OVF/OVA file at the end), Packer sometimes leverages OVF Tool for other interactions or conversions. It's generally good practice to have it installed.

```bash
https://developer.broadcom.com/tools/open-virtualization-format-ovf-tool/latest
chmod +x ovf
sudo mv ovf /usr/local/bin/ovftool
```

Verify the installation:

```bash
ovftool --version
```

## Build the Virtual Machine with Packer

Now you can run Packer, pointing it to your JSON configuration file. Packer will connect to your ESXi host, copy the template files, modify the VMX settings, boot the new VM, run the provisioners, and finally shut it down.

```bash
packer build your-config-file.json
```

(Replace your-config-file.json with the actual name of your Packer configuration file)

### Example Packer Configuration

This example demonstrates building a Debian 11 VM named "Zakops" based on an existing template. (your-config-file.json)

```bash
{
  "builders": [
    {
      // --- Builder Type ---
      "type": "vmware-vmx", // Uses an existing VMX file as a base

      // --- Source VM Template ---
      // Path to the source VMX relative to the 'remote_datastore'
      "source_path": "New Template [Debian12]/New Template [Debian12].vmx",
      // Alternative: If source is on a *different* datastore, use relative path hack
      // "source_path": "../Datastore-Raid5/Debian12/Debian12.vmx",

      // --- New VM Settings ---
      "vm_name": "Zakops",      // Name for the new VM folder and files on the datastore
      "vmx_data": {                // Overrides or adds parameters to the new VMX file
        "displayName": "Zakops-Machine", // Name displayed in vSphere/ESXi inventory
        "memsize": "4096",           // RAM in MB (e.g., 4GB)
        "numvcpus": "2",             // Number of virtual CPUs
        "guestOS": "debian11-64"     // VMware Guest OS Identifier (important for optimizations)
        //"annotation": "Built by Packer - Debian 11 Base" // Optional descriptive note
      },

      // --- ESXi Connection Settings ---
      "remote_type": "esx5",           // Connect directly to an ESXi host
      "remote_host": "Vmware-IP",      // IP address or hostname of your ESXi host
      "remote_username": "root",       // ESXi username (use a dedicated user if possible)
      "remote_password": "P@$w0rD",    // ESXi password
      "remote_datastore": "Datastore2", // Target datastore for the new VM
      "insecure_connection": true,     // Disables SSL certificate verification (use with caution)

      // --- Guest VM Connection (SSH) ---
      "ssh_username": "USER",              // Username inside the guest VM for provisioning
      "ssh_host": "Zakops-Machine-IP",          // *Target* IP for SSH (VM needs to acquire this IP, often via DHCP first or set by a provisioner)
      "ssh_port": 4422,                  // SSH port on the guest VM
      "ssh_private_key_file": "/root/.ssh/id_ed25519", // Path to SSH private key for guest authentication
      "ssh_certificate_file": "/root/.ssh/vault.pub", // Optional: Path to SSH certificate for signed key auth
      "ssh_timeout": "3m",               // Timeout for SSH connection to the guest
      "ssh_handshake_attempts": 5,       // Number of SSH connection retries
      "ssh_clear_authorized_keys": true, // Removes Packer's temporary key after provisioning

      // --- Build Process Options ---
      "headless": true,                  // Run VM without a console window
      "disable_vnc": true,               // Disable VNC access during build
      "skip_export": true,               // Do not export the VM as OVF/OVA after build
      "keep_registered": true,           // Keep the VM registered in ESXi after build

      // --- Guest Shutdown ---
      "shutdown_command": "echo 'packer' | sudo -S shutdown -h now" // Command to shut down the guest VM gracefully
    }
  ],
  "provisioners": [
    {
      // --- Example Provisioner: Update Guest OS ---
      "type": "shell",
      "inline": [
        "echo 'Running system updates...'",
        "sudo apt update && sudo apt upgrade -y",
        "echo 'System updates completed.'"
        // Add more provisioning steps here (e.g., install software, configure services, set static IP)
      ]
    }
  ]
}
```

Explanation of Key Configuration Parameters:

* type: "vmware-vmx": Specifies that you are starting from an existing .vmx file template.

* source\_path: Location of the source .vmx file. Crucially, paths are interpreted differently based on whether the source and destination datastores are the same. If they are the same (remote\_datastore), the path is relative to that datastore. If they are different, you might need the ../OtherDatastore/path/to/vm.vmx trick.

* vm\_name: The name used for the VM's folder and files (.vmx, .vmdk, etc.) on the target datastore.

* vmx\_data: Allows you to inject or override settings within the .vmx file of the new VM being created. displayName is what you see in the vSphere inventory.

* remote\_host, remote\_username, remote\_password: Credentials for connecting to the ESXi host itself.

* remote\_datastore: The ESXi datastore where the new virtual machine will be created.

* ssh\_host: Important: This is the IP address Packer expects the guest VM to have when it tries to connect via SSH for provisioning. If your template uses DHCP, Packer (with VMware Tools installed in the guest) will usually find the initial DHCP address, connect, and then you might use a provisioner to set a static IP. If you set a static IP, ensure it matches this value.

* skip\_export & keep\_registered: Setting both to true means Packer builds the VM, provisions it, shuts it down, and leaves it registered on the ESXi host without creating an OVF/OVA export file.

* provisioners: This section defines the steps Packer executes inside the guest VM after booting it (e.g., running shell scripts, using Ansible, Chef, etc.).

This setup provides a solid foundation for automating your VMware VM builds using Packer directly against an ESXi host. Remember to replace placeholder values in the JSON example with your actual environment details.

