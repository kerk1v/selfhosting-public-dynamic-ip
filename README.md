# selfhosting-public-dynamic-ip
How to self-host services on a Homeserver or Homelab and expose them to the public internet (despite your ISP making this as difficult as possible). 

## The problem: 

You have some services running on your Homeserver or Homelab? You have a decent Internet connection with high bandwidth but your ISP doesn't give you a static IP address, so you're either stuck with a dynamic IP and an DynDNS-like service. Or even worse, your ISP uses CGNAT (Carrier Grade Network Address Translation) so you don't have a public, routable IP address at all? 

Here in Spain we heve pretty decent fibre connections, 1Gbit/s symmetrical is quite frequent, 300/300Mbit/s is considered low-end, I have 10/10Gbit/s (with net data rates around 8Gbit/s) from DIGI, but a fixed IP is either not available or costs more money than the solution explained here. It would be shame let so much bandwidth go to waste. You could even host a small personal website or blog at home. All this is possible with this solution. 

You might ask, I can use Dynamic DNS and Port forwarding, why shouldn't I do that? Well, this opens a portion, however small it could be of your home network to the Internet. Misconfigure it, and you might expose the wrong PC or some vulnerable part of your network. The proposed solution exposes only the machine at home you specifically choose, so you can put "public" apps on that machine, and keep the rest of your home network locked and reasonably secure.  

Fear no more, as you can make your services - and only the ones you chose - available from the internet for a handful of €uros a month. With automatic SSL/HTTPS with Acme / LetsEncrypt as a bonus.  

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
8. Check that the connection is established with the command ```ip a``` you should see something similar to this at the end: 
    ```tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.8.0.10/24 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fd42:42:42:42::2/112 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::d652:3673:e6ac:2d2b/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever```
    This means that you have sucessfully connected to the Cloud server.
9. You can also test connectivity with the Cloud server with ```ping 10.8.0.1```. It should respond.

### Back to the Cloud Server

1. Test connectivity to the Homeserver: ```ping 10.8.0.10```. It should also respond. The ```tun0``` network interface should also be present if you check with ```ip a```.
2. Install Docker on the Cloud server. I follow the official instructions here: https://docs.docker.com/engine/install/ubuntu/
3. Install npm on the Cloud server: ```apt install npm -y```.
4. Install CapRover on the Cloud server. Caprover is an easy-to-use PAAS (Platform-As-A-Service) that runs even on the most modest servers. I use it because it provides a lot of functions, like automatic SSL/HTTPS with free LetsEncrypt / ACME certificates. Read more about it here: https://caprover.com/
```docker run -p 80:80 -p 443:443 -p 3000:3000 -e ACCEPTED_TERMS=true -v /var/run/docker.sock:/var/run/docker.sock -v /captain:/captain caprover/caprover```
5. In the meantime, install the CapRover cli: ```npm install -g caprover```.
6. Wait for a bit for CapRover to come up.
7. Have the IP address of your Cloud server ready. Do the initial configuration of CapRover with the command ```caprover serversetup```. Answer all the questions. As the root domain enter <yourdomain.com>. Choose a secure password when prompted, and wait for about a minute after the command has completed.
8. Navigate to https://captain.yourdomain.com. You should see this screen:
![Screenshot 2023-12-18 at 18 42 45](https://github.com/kerk1v/selfhosting-public-dynamic-ip/assets/16302524/e94a0398-4a3b-4435-990e-5b4df11ffdc2)
9. If you successfully log in with the password that you set during step 7, you will see this screen:
![Screenshot 2023-12-18 at 18 53 48](https://github.com/kerk1v/selfhosting-public-dynamic-ip/assets/16302524/03d1d87a-d52e-4536-a022-9e1b61715f4c)


OK, done here. Back to the Homeserver. 

### At the Homeserver again

The first thing you need is a web application to share. Let's assume, for simplicity that you have a Grafana instance running on your homeserver on port 3000 (the default) and want to access that Grafana instance from anywhere in the world. Make sure it's running and really works on that port. 

### Back to the CapRover web interface. 

1. On the sidabar of the web interface, click on "Apps"
![Screenshot 2023-12-18 at 18 45 56](https://github.com/kerk1v/selfhosting-public-dynamic-ip/assets/16302524/e8844ada-59f5-49b5-b33e-9cffaf4c9c5a)
2. Click the "One-Click Apps/Databases" button.
3. Select the "Nginx Reverse Proxy" app from the list that comes up
![Screenshot 2023-12-18 at 18 57 28](https://github.com/kerk1v/selfhosting-public-dynamic-ip/assets/16302524/5474ac20-6b22-49e7-b3cf-520f176f2606)
4. Enter the required data in the next dialog:
![Screenshot 2023-12-18 at 18 58 53](https://github.com/kerk1v/selfhosting-public-dynamic-ip/assets/16302524/4b167624-dc7c-4f6d-a409-92c3b58644c1)
    1.  App Name: This will be the hostname part under which your app will be available, the complete URL will be https://<appname>.<yourdomain.com>. If you're not using a wildcard DNS entry, make sure you create the approprate DNS entry before hitting "Deploy" in step 4 below!
    2.  Version: leave this field alone for now. It is the version of the image from Dockerhub that will be used.
    3.  Upstream address: This will always start with http://10.8.0.10 (the IP of the Homeserver on the OpenVPN connection) followed by the port number it's listening on (in our case, :3000).
    4.  Click "Deploy" at the bottom right of the page, and wait until it completes.
    5.  Now navigate to the "Apps" screen again. Your newly created app will be there. Click on the name, we have two more settings to adjust.
    6.  Click on the "Enable HTTPS" button. After some seconds, the button will be grayed out and the URL next to it will have changed to start with https://.
    7.  Enable the checkbox "Force HTTPS by redirecting all HTTP traffic to HTTPS" and hit the "Save and Restart" button. You're done. Your Grafana app should now be accessible from the internet as https://grafana.<yourdomain.com>.
  
Proceed in the same way with any service running on your Homeserver or Homelab. The proposed Cloud server will easily handle dozens of them, the NginX image used is very lightweight and does not use much memory. As I said, if you outperform this server or run over the traffic limit of Hetzner Cloud, this is not the right solution for you. 

In case you run into problems, please feel free to open an issue here on GitHub. I will do my best to help. For issues with CapRover please refer to their documentation at https://caprover.com/docs/get-started.html or their github repository / issues at https://github.com/caprover/caprover/issues. 

Suggestions for improvements? Fork the repo, make your changes and create a Pull Request, or write in the "Issues" section. 
