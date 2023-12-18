# selfhosting-public-dynamic-ip
How to self-host services on a Homeserver or Homelab and expose them to the public internet (despite your ISP making this as difficult as possible). 

## The problem: 

You have a load of services running on your Homeserver or Homelab? You have a decent Internet connection with high bandwidth but your ISP doesn't give you a static IP address, so you're either stuck with a dynamic IP. Or even worse, your ISP uses CGNAT (Carrier Grade Network Address Translation) so you don't have a public, routable IP address at all?  

Fear no more, as you can make your services - and only the ones you chose - available from the internet for a handful of €uros a month. With automatic SSL/HTTPS as a bonus, and 

## What do I need? 

- Of course, a Homeserver or Homelab with services you want to make available on the public internet.

- A cheap hosted server or VM. I am personally very fond of Hetzner Cloud, a german no-frills but very reliable hosting and cloud provider. You can get their most affordable ARM cloud instances for less than 5€ a month (price chages depending on your national VAT rate) and get roughly 4 months for free (20€ Cloud credit) if you sign up with them using my referral link: https://hetzner.cloud/?ref=uuMclGJHJMf4. Once you spend your free credit and use 10 additional Euros of their services, I get 10€ free credit. Thanks in advance!

- An Internet Domain name, and the ability to set DNS records for it. 

- Some basic Linux CLI knowledge. Not much, as I will explain everyting in great detail below.

## Enough, I want to get this done now! 

Good, good, here are the steps, one by one...

I am assuming that you will run all commands as root or with ```sudo``` to have appropriate privileges. 

### Steps to get and do the initial configuration on the Cloud server.  

1. Get the cloud instance mentioned above. If you decide to go with Hetzner, the ARM CAX11 with 2 VCPU, 4GB of RAM and 40 GB of SSD storage will be more than enough for our purposes. They have a limitation of 20TB of traffic per month, this should only be relevant for sites with very heavy traffic, in which case you should be using a different approach anyways. Some hints for creating the machine.
    1. I use Ubuntu 22.04 as my OS, so the instructions below are tested and will work with that OS. 
    2. Add an IPv4 address. The state of IPv6 unfortunately is less than optimal, and many ISPs aren't giving out IPv6 addresses to their clients (yet).
    3. Set a hostname of captain.domain.com. More on that later. 
    4. Use an SSH-Key for logging in to the cloud instance. Not only is this more secure than password authetication, it's also much more convenient.
        1. On OSX and Linux, check the directory  ~/.ssh for two files named `id_rsa` and `id_rsa.pub`. If they are there, you're good to go. If not, use the command ```ssh-keygen``` to generate a keypair (these two files)
        2. On Windows (10 and 11) you need to install the additional component called "SSH Client". After that, pretty much everything is the same as above.
        3. Upload the PUBLIC key into the Hetzner cloud server creation interface. Paste the contents of ``ìd_rsa.pub``` into that field, give it a memorable name and if you wish, check the box to make it the default key.
        4. Don't forget to save the keypair (```id_rsa``` and ```id_rsa.pub```) in a safe place in case you ever format your PC. 
2. Once the cloud instance is created, get to your DNS panel at your DNS provider and set up a DNS record pointing to the IP address of the cloud server you've just created. I use a wildcard entry, pointing any request to *.domain.com to the cloud instance, but for starters set up at least captain.domain.com. More on that later
3. Login to the Cloud server you heve just created, normally the command will be ```ssh root@captain.domain.com```.
4. Run the commands ```apt update && apt upgrade -y``` to apply the latest updates to the OS. Remember to do that once in a while to keep your OS up-to-date with the latest security patches. 
5. Run the following command to get the OpenVPN server install script: ```wget https://raw.githubusercontent.com/Angristan/openvpn-install/master/openvpn-install.sh -O ubuntu-22.04-lts-vpn-server.sh```.
6. Make the script executable with ```chmod +x ubuntu-22.04-lts-vpn-server.sh```.
7. Execute the script with the following command: ```./ubuntu-22.04-lts-vpn-server.sh```. It's fine to keep the default values. Don't enable compression.
8. At the end of the script execution, it will ask you for a client name. Use something easy to remember, like Homeserver or Homelab. Use Passwordless authetication. More convenient in the scenatio we will be using, and also more secure.
9. Now issue the command ```echo 'ifconfig-push 10.8.0.10 255.255.255.0' >> /etc/openvpn/ccd/<client name from step 8> ```. This will make sure our OpenVPN client (our Homeserver) will always get the same IP when connecting to the Cloud server.
10. run ```systemctl restart openvpn@server.service``` so that the server picks up the new configuration
11. Find a file with the extension ```.ovpn``` in the current directory. Display that file with ```cat <filename>``` and keep the content somewhere (open terminal window, text editor, whatever is more convenient for you) 

### Steps to on the Homeserver

1. Install the OpenVPN package. For Ubuntu the command will be ```apt install openvpn -y```.
2. Edit the file ```nano /etc/openvpn/client.conf``` with your favorite editor. I use nano, but please, feel free to use whatever you prefer.
3. Paste the content of the file from step 11 above into this file and save it
4. Insert the line ```pull-filter ignore redirect-gateway``` somewhere in the first section of the file, before the line ```<ca>```. This line is needed so that normal Internet traffic is still routed via your local router and not through the Cloud Server
5. Edit the file ```/etc/default/openvpn```. Uncomment the line ```AUTOSTART="all"```.
6. Test the connection with the command ```openvpn --config /etc/openvpn/client.conf```. If you see ```Initialization Sequence Completed``` at the end, all is fine. Exit with ctrl+c.
7. Now run the commands ```systemctl daemon-reload``` and ```systemctl start openvpn```.
8. Check that the connection is established with the command ```ip a``` you should see eomething similar to this at the end: 
    ```tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.8.0.10/24 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fd42:42:42:42::2/112 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::d652:3673:e6ac:2d2b/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever```

