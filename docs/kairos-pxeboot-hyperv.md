<!-----

Yay, no errors, warnings, or alerts!

Conversion time: 0.422 seconds.


Using this Markdown file:

1. Paste this output into your source file.
2. See the notes and action items below regarding this conversion run.
3. Check the rendered output (headings, lists, code blocks, tables) for proper
   formatting and use a linkchecker before you publish this page.

Conversion notes:

* Docs to Markdown version 1.0β34
* Fri Nov 03 2023 04:50:02 GMT-0700 (PDT)
* Source doc: Copy of Kairos Install on Hyper-v
----->



# Installing Kairos via PXE Boot on Microsoft Hyper-V

The following procedure can be followed to create a Kairos based Kubernetes (k3s) cluster with high-availability control plane, suitable to form the basis of a Mojaloop hosting environment.



1. If not already present, add the “hyper-v” role to the windows server OS being used to host VMs (necessary for all server nodes hosting VMs).
2. Create a new Internal hyper-v virtual switch
    1. Give it a sensible name e.g. “Internal NAT Switch”
    2. Follow the instructions here to make the switch you just created act as a NAT device (set the NAT switch with 192.168.0.1 as its IPv4 address): [https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network)
3. Create an AuroraBoot PXE boot server:
    3. Download ubuntu 20.04 LTS ISO from [https://ubuntu.com/download/server](https://ubuntu.com/download/server)
    4. Create a new hyper-v VM
        1. Set it to have 2048MB RAM, 2 VCPU, 50GB HDD.
        2. Set it to boot from DVD drive
        3. Set the downloaded Ubuntu  Server ISO as the DVD drive contents
        4. Disable secure boot
        5. Connect its network adapter to the newly created NAT internal virtual switch
    5. Boot the VM and follow the prompts to install Ubuntu server onto the HDD.
        6. Set the network interface to have a static IPv4 address of 192.168.0.2 netmask 255.255.255.0/24
    6. Install a DHCP server (from [https://www.howtoforge.com/how-to-install-and-configure-dhcp-server-on-ubuntu-20-04/](https://www.howtoforge.com/how-to-install-and-configure-dhcp-server-on-ubuntu-20-04/))
        7. Run: `sudo apt-get install isc-dhcp-server -y`
        8. Run: `sudo systemctl start isc-dhcp-server`
        9. Run: `sudo systemctl enable isc-dhcp-server`
        10. Configure the DHCP server to grant leases on the 192.168.0.x subnet:
            1. Edit (e.g. using vi) `/etc/dhcp/dhcpd.conf`
                1. Uncomment “`authoritative"`
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
        11. Check the DHCP server is running:
            3. Run: `sudo systemctl status isc-dhcp-server`
    7. Install docker (from [https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04)):
        12. Run: `sudo apt update`
        13. Run: `sudo apt install apt-transport-https ca-certificates curl software-properties-common`
        14. Run: `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`
        15. Run: `sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"`
        16. Run: `apt-cache policy docker-ce`
        17. Run: `sudo apt install docker-ce`
        18. Run: `sudo systemctl status docker`
    8. Make a **control plane node **cloud config file:
        19. Generate an SSH keypair:
            4. Run: `ssh-keygen`
            5. Accept the defaults by pressing the “enter” key (do not enter a passphrase)
        20. Copy the contents of the newly generated SSH public key to the clipboard:
            6. The file should be found at: `~/.ssh.id_rsa.pub`
        21. Create a text file (e.g. with vi) called `config.yaml` with the following content. (_Paste the contents of the SSH public key file in the specified place below_):

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
              - ——cluster-init
              env:
                K3S_TOKEN: ""
            ```


    9. Run the Kairos AuroraBoot server as a docker container:
        22. Run: `sudo docker run -v /home/bushj/config.yaml:/config.yaml --rm -ti --net host quay.io/kairos/auroraboot:v0.2.4 --set container_image=quay.io/kairos/kairos-alpine-ubuntu:v2.0.3-k3sv1.21.14-k3s1 --cloud-config=/config.yaml`
    10. Wait for the required images to download and the kairos ISO to be built and served by AuroraBoot:
        23. When you see the following output on the console you will know that AuroraBoot is ready to serve PXE boot requests:

            ```
            Listening on :8080…
            ```


4. Create a k3s node VM (repeat this step for each cluster VM required):
    11. Create a new hyper-v VM
        24. Set it to have 4096MB RAM, 2 VCPU, 50GB (new, empty) HDD.
        25. Disable secure boot
        26. Set it to boot from HDD first, then network adapter second.
        27. Connect its network adapter to the newly created NAT internal virtual switch
    12. Boot the new VM.
        28. It should successfully download boot artefacts via PXE boot, install the Kairos operating system and reboot from its HDD to a stable k3s enabled kairos node.

            		
