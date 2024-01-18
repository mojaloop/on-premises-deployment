# Installing Kairos via PXE Boot on Microsoft Hyper-V

The following procedure can be followed to create a Kairos based Kubernetes (k3s) cluster with high-availability control plane, suitable to form the basis of a Mojaloop hosting environment.

1. If not already present, add the “hyper-v” role to the windows server OS being used to host VMs (necessary for all server nodes hosting VMs).
2. Create a new Internal hyper-v virtual switch
    1. Give it a sensible name e.g. “Internal NAT Switch”
    2. Follow the instructions here to make the switch you just created act as a NAT device (set the NAT switch with 192.168.0.1 as its IPv4 address): [https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network)
3. Create an AuroraBoot PXE boot server:
    1. Download ubuntu 20.04 LTS ISO from [https://ubuntu.com/download/server](https://ubuntu.com/download/server)
    2. Create a new hyper-v VM
        1. Set it to have 2048MB RAM, 2 VCPU, 50GB HDD.
        2. Set it to boot from DVD drive
        3. Set the downloaded Ubuntu  Server ISO as the DVD drive contents
        4. Disable secure boot
        5. Connect its network adapter to the newly created NAT internal virtual switch
    3. Boot the VM and follow the prompts to install Ubuntu server onto the HDD.
        1. Set the network interface to have a static IPv4 address of 192.168.0.2 netmask 255.255.255.0/24
    4. Install a DHCP server (from [https://www.howtoforge.com/how-to-install-and-configure-dhcp-server-on-ubuntu-20-04/](https://www.howtoforge.com/how-to-install-and-configure-dhcp-server-on-ubuntu-20-04/))
       1. Run: `sudo apt-get install isc-dhcp-server -y`
          1. Run: `sudo systemctl start isc-dhcp-server`
          2. Run: `sudo systemctl enable isc-dhcp-server`
          3. Configure the DHCP server to grant leases on the 192.168.0.x subnet:
             1. Edit (e.g. using vi) `/etc/dhcp/dhcpd.conf`
                1. Uncomment `authoritative`
                2. Change `max-lease-time` to `6300`
                3. Add a subnet thus:
              
                    ```
                    subnet 192.168.0.0 netmask 255.255.255.0 {
                        range 192.168.0.2 192.168.0.200;
                        option routers 192.168.0.1;
                        option domain-name-servers 8.8.8.8, 8.8.4.4;
                    }
                    ```
    
             2. Run: sudo `systemctl restart isc-dhcp-server`
       2. Check the DHCP server is running:
          1. Run: `sudo systemctl status isc-dhcp-server`
    5. Install docker (from [https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04)):
        1. Run: `sudo apt update`
        2. Run: `sudo apt install apt-transport-https ca-certificates curl software-properties-common`
        3. Run: `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`
        4. Run: `sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"`
        5. Run: `apt-cache policy docker-ce`
        6. Run: `sudo apt install docker-ce`
        7. Run: `sudo systemctl status docker`
    6. Make a **control plane node** cloud config file:
        1. Generate an SSH keypair:
           1. Run: `ssh-keygen`
           2. Accept the defaults by pressing the “enter” key (do not enter a passphrase)
        2. Copy the contents of the newly generated SSH public key to the clipboard:
           1. The file should be found at: `~/.ssh.id_rsa.pub`
        3. Create a text file (e.g. with vi) called `config.yaml` with the following content, and note its path. (_Paste the contents of the SSH public key file in the specified place below_):

           ```
           #cloud-config
           # this is the config file for k3s control plane nodes

           install:
             auto: true
             device: "auto"
             reboot: true

           users:
           - name: "kairos"
             passwd: "kairos"
             ssh_authorized_keys:
             - "{{ PASTE SSH PUBLIC KEY HERE }}"

           k3s:
             enabled: true
             args:
             - --cluster-init
             env:
               K3S_TOKEN: ""
           ```

    7. Run the Kairos AuroraBoot server as a docker container (_Paste the path to your config.yaml file in the specified place below_)
        1. Run: `sudo docker run -v {{ INSERT FULL PATH TO config.yaml FILE HERE }}config.yaml:/config.yaml --rm -ti --net host quay.io/kairos/auroraboot:v0.2.4 --set container_image=quay.io/kairos/kairos-alpine-ubuntu:v2.0.3-k3sv1.21.14-k3s1 --cloud-config=/config.yaml`
    8. Wait for the required images to download and the kairos ISO to be built and served by AuroraBoot:
        1. When you see the following output on the console you will know that AuroraBoot is ready to serve PXE boot requests:

            ```
            Listening on :8080…
            ```

4. Create a k3s node VM (repeat this step for each cluster VM required):
    1. Create a new hyper-v VM
       1. Set it to have 4096MB RAM, 2 VCPU, 50GB (new, empty) HDD.
       2. Disable secure boot
       3. Set it to boot from HDD first, then network adapter second.
       4. Connect its network adapter to the newly created NAT internal virtual switch
    2. Boot the new VM.
       1. It should successfully download boot artefacts via PXE boot, install the Kairos operating system and reboot from its HDD to a stable k3s enabled kairos node.

            		
## TODOs
- Add templating to the config.yaml cloud config file:
  - Allow for specified number of control plane nodes with different passwords etc...
  - Allow for specified number of worker nodes with different passwords etc...
- Add instructions for adding worker nodes to the cluster