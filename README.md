# selfhosting-public-dynamic-ip
How to self-host services on a Homeserver or Homelab and expose them to the public internet (despite your ISP making this as difficult as possible). 

### The problem: 

You have a load of services running on your Homeserver or Homelab? You have a decent Internet connection with high bandwidth but your ISP doesn't give you a static IP address, so you're either stuck with a dynamic IP. Or even worse, your ISP uses CGNAT (Carrier Grade Network Address Translation) so you don't have a public, routable IP address at all?  

Fear no more, as you can make your services - and only the ones you chose - available from the internet for a handful of €uros a month. With automatic SSL/HTTPS as a bonus, and 

### What do I need? 

- Of course, a Homeserver or Homelab with services you want to make available on the public internet.

- A cheap hosted server or VM. I am personally very fond of Hetzner Cloud, a german no-frills but very reliable hosting and cloud provider. You can get their most affordable ARM cloud instances for less than 5€ a month (price chages depending on your national VAT rate) and get roughly 4 months for free (20€ Cloud credit) if you sign up with them using my referral link: https://hetzner.cloud/?ref=uuMclGJHJMf4. Once you spend your free credit and use 10 additional Euros of their services, I get 10€ free credit. Thanks in advance!

- An Internet Domain name, and the ability to set DNS records for it. 

- Some basic Linux CLI knowledge. Not much, as I will explain everyting in great detail below.

### Enough, I want to get this done now! 

Good, good, here are the steps, one by one...

1. Get the cloud instance mentioned above. If you decide to go with Hetzner, the ARM CAX11 with 2 VCPU, 4GB of RAM and 40 GB of SSD storage will be more than enough for our purposes. Some hints for creating the machine.
    1. I use Ubuntu 22.04 as my OS, so the instructions below are tested and will work with that OS. 
    2. Add an IPv4 address. The state of IPv6 unfortunately is less than optimal, and many ISPs aren't giving out IPv6 addresses to their clients (yet).
    3. Set a hostname of captain.domain.com. More on that later. 
    4. Use an SSH-Key for logging in to the cloud instance. Not only is this more secure than password authetication, it's also much more convenient.
        1. On OSX and Linux, check the directory  ~/.ssh for two files named `id_rsa` and `id_rsa.pub`. If they are there, you're good to go. If not, use the command ```ssh-keygen``` to generate a keypair (these two files)
        2. On Windows (10 and 11) you need to install the additional component called "SSH Client". After that, pretty much everything is the same as above.
        3. Upload the PUBLIC key into the Hetzner cloud server creation interface. Paste the contents of ``ìd_rsa.pub``` into that field, give it a memorable name and if you wish, check the box to make it the default key.
        4. Don't forget to save the keypair (``ìd_rsa``` and ``ìd_rsa.pub```) in a safe place in case you ever format your PC. 
2. Once the cloud instance is created, get to your DNS panel at your DNS provider and set up a DNS record pointing to the IP address of the cloud server you've just created. I use a wildcard entry, pointing any request to *.domain.com to the cloud instance, but for starters set up at least captain.domain.com. More on that later
3. Login to the Cloud server you heve just created, normally the command will be ```ssh root@captain.domain.com```.
4. Run the commands ```apt update && apt upgrade -y``` to apply the latest updates to the OS. Remember to do that once in a while to keep your OS up-to-date with the latest security patches. 
5. Run the following command to get the OpenVPN server install script: ```wget https://raw.githubusercontent.com/Angristan/openvpn-install/master/openvpn-install.sh -O ubuntu-22.04-lts-vpn-server.sh```.
6. Make the script executable with ```chmod +x ubuntu-22.04-lts-vpn-server.sh```.
7. Execute the script with the following command: ```./ubuntu-22.04-lts-vpn-server.sh```. It's fine to keep the default values. Don't enable compression.
8. At the end of the script execution, it will ask you for a client name. Use something easy to remember, like Homeserver or Homelab. 
