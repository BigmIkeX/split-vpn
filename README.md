# split-vpn
A split tunnel VPN script for the UDM with policy based routing.

## What is this?

This is a helper script for the OpenVPN or WireGuard client on the UDM that creates a split tunnel for the VPN connection, and forces configured clients through the VPN instead of the default WAN. This is accomplished by marking every packet of the forced clients with an iptables firewall mark (fwmark), adding the VPN routes to a custom routing table, and using a policy-based routing rule to direct the marked traffic to the custom table. 

## Features

* Force traffic to the VPN based on source interface (VLAN), MAC address, IP address, or IP sets.
* Exempt sources from the VPN based on IP, MAC address, IP:port, MAC:port combinations, or IP sets. This allows you to force whole VLANs through by interface, but then selectively choose clients from that VLAN, or specific services on forced clients, to exclude from the VPN.
* Exempt destinations from the VPN by IP. This allows VPN-forced clients to communicate with the LAN or other VLANs.
* Force domains to the VPN or exempt them from the VPN (only supported with dnsmasq or pihole). 
* Port forwarding on the VPN side to local clients (not all VPN providers give you ports).
* Redirect DNS for VPN traffic to either an upstream DNS server or a local server like pihole, or block DNS requests completely.
* Built-in kill switch via iptables and blackhole routing.
* Works across IP changes and network restarts. 
* Can be used with multiple openvpn instances with separate configurations for each. This allows you to force different clients through different VPN servers. 
* IPv6 support for all options.
* Run on boot support via UDM-Utilities boot script.
* Supports OpenVPN, WireGuard kernel module, and wireguard-go docker container.

## Compatibility

This script is designed to be run on the UDM-Pro and UDM base. It has been tested on 1.8.x, 1.9.x, and 1.10.x, however other versions should work. Please submit a bug report if you use this on a different version and encounter issues. 

## How do I use this?

<details>
  <summary>Click here to see the instructions for OpenVPN.</summary>

1. SSH into the UDM/P (assuming it's on 192.168.1.254).

    ```sh
    ssh root@192.168.1.254
    ```
    
2. Download the scripts package and extract it to `/mnt/data/split-vpn/vpn`.

    ```sh
    cd /mnt/data
    mkdir /mnt/data/split-vpn && mkdir /mnt/data/split-vpn/vpn
    cd /mnt/data/split-vpn
    curl -L https://github.com/peacey/split-vpn/archive/main.zip | unzip - "*/vpn/*" -o -j -d vpn && chmod +x vpn/*.sh
    ```
    
3. Create a directory for your VPN provider's openvpn configuration files, and copy your VPN's configuration files (certificates, config, password files, etc) and the sample vpn.conf from `/mnt/data/split-vpn/vpn/vpn.conf.sample`. NordVPN is used below as an example. 

    ```sh
    mkdir -p /mnt/data/split-vpn/openvpn/nordvpn
    cd /mnt/data/split-vpn/openvpn/nordvpn
    curl https://downloads.nordcdn.com/configs/files/ovpn_legacy/servers/us-ca12.nordvpn.com.udp1194.ovpn --out nordvpn.ovpn
    cp /mnt/data/split-vpn/vpn/vpn.conf.sample /mnt/data/split-vpn/openvpn/nordvpn/vpn.conf
    ```
    
4. If your VPN provider uses a username/password, put them in a `username_password.txt` file in the same directory as the configuration with the username on the first line and password on the second line. Then either: 
    * Edit your VPN provider's openvpn config you downloaded in step 3 to reference the username_password.txt file by adding/changing this directive: `auth-user-pass username_password.txt`.
    * Use the `--auth-user-pass username_password.txt` option when you run openvpn below in step 6 or 8. 
    
    NOTE: The username/password for openvpn are usually given to you in a file or in your VPN provider's online portal. They are usually not the same as your login to the VPN. 
5. Edit the `vpn.conf` file with your desired settings. See the explanation of each setting [below](#configuration-variables). 
6. Run OpenVPN in the foreground to test if everything is working properly.

    ```sh
    openvpn --config nordvpn.ovpn \
            --route-noexec \
            --up /mnt/data/split-vpn/vpn/updown.sh \
            --down /mnt/data/split-vpn/vpn/updown.sh \
            --script-security 2
    ```
    
7. If the connection works, check each client to make sure they are on the VPN by doing the following.

    * Check if you are seeing the VPN IPs when you visit http://whatismyip.host/. You can also test from command line, by running the following commands from your clients. Make sure you are not seeing your real IP anywhere, either IPv4 or IPv6.
    
      ```sh
      curl -4 ifconfig.co
      curl -6 ifconfig.co
      ```
        
      If you are seeing your real IPv6 address above, make sure that you are forcing your client through IPv6 as well as IPv4, by forcing through interface, MAC address, or the IPv6 directly. If IPv6 is not supported by your VPN provider, the IPv6 check will time out and not return anything. You should never see your real IPv6 address. 

    * Check for DNS leaks with the Extended Test on https://www.dnsleaktest.com/. If you see a DNS leak, try redirecting DNS with the `DNS_IPV4_IP` and `DNS_IPV6_IP` options, or set `DNS_IPV6_IP="REJECT"` if your VPN provider does not support IPv6. 
    * Check for WebRTC leaks in your browser by visiting https://browserleaks.com/webrtc. If WebRTC is leaking your IPv6 IP, you need to disable WebRTC in your browser (if possible), or disable IPv6 completely by disabling it directly on your client or through the UDMP network settings for the client's VLAN.
    
8. If everything is working properly, stop the OpenVPN client by pressing Ctrl+C, and then run it in the background with the following command. If you want to enable the killswitch to block Internet access to forced clients if OpenVPN crashes, set `KILLSWITCH=1` in the `vpn.conf` file before starting OpenVPN. If you also want to block Internet access to forced clients when you exit OpenVPN cleanly (with SIGTERM), then set `REMOVE_KILLSWITCH_ON_EXIT=0`.

    ```sh
    nohup openvpn --config nordvpn.ovpn \
                  --route-noexec \
                  --up /mnt/data/split-vpn/vpn/updown.sh \
                  --down /mnt/data/split-vpn/vpn/updown.sh \
                  --script-security 2 \
                  --ping-restart 15 \
                  --mute-replay-warnings &> openvpn.log &
    ```
    You can modify the command to change `--ping-restart` or other options as needed. The only requirement is that you run updown.sh script as the up/down script and `--route-noexec` to disable OpenVPN from adding routes to the default table instead of our custom one.
    
9. Now you can exit the UDM/P. If you would like to start the VPN client at boot, please read on to the next section. 
10. If your VPN provider doesn't support IPv6, it is recommended to disable IPv6 for that VLAN in the UDMP settings, or on the client, so that you don't encounter any delays. If you don't disable IPv6, clients on that network will try to communicate over IPv6 first and fail, then fallback to IPv4. This creates a delay that can be avoided if IPv6 is turned off completely for that network or client.

</details>

<details>
  <summary>Click here to see the instructions for WireGuard (kernel module).</summary>

  * **Prerequisuite:** Make sure the WireGuard kernel module is installed via either [wireguard-kmod](https://github.com/tusc/wireguard-kmod) or a [custom kernel](https://github.com/fabianishere/udm-kernel-tools). The WireGuard tools (wg-quick, wg) also need to be installed (included with wireguard-kmod) and accessible from your PATH.
  * Make sure you run the wireguard setup script from [wireguard-kmod](https://github.com/tusc/wireguard-kmod) once, then test the installation of the module by running `modprobe wireguard` which should return nothing and no errors, and running `wg-quick` which should return the help and no errors. 
  * Note that the kernel module is dependent on the software version of your UDMP because each software update usually brings a new kernel version. If you update the UDM/P software, you also need to update the kernel module to the new version once it is released (or compile your own module for the new kernel). The module will fail to run on a kernel it was not compiled for. Hence, you have to be careful that the UDMP doesn't perform a sofware update unexpectedly if you use this module. 
  
1. SSH into the UDM/P (assuming it's on 192.168.1.254).

    ```sh
    ssh root@192.168.1.254
    ```
  
2. Download the scripts package and extract it to `/mnt/data/split-vpn/vpn`.

    ```sh
    cd /mnt/data
    mkdir /mnt/data/split-vpn && mkdir /mnt/data/split-vpn/vpn
    cd /mnt/data/split-vpn
    curl -L https://github.com/peacey/split-vpn/archive/main.zip | unzip - "*/vpn/*" -o -j -d vpn && chmod +x vpn/*.sh
    ```
    
3. Create a directory for your WireGuard configuration files, copy the sample vpn.conf from `/mnt/data/split-vpn/vpn/vpn.conf.sample`, and copy your WireGuard configuration file (wg0.conf) or create it. As an example below, we are creating the wg0.conf file that mullvad provides and pasting the contents into it. You can use any name for your config instead of wg0 (e.g.: mullvad-ca2.conf) and this will be the interface name of the wireguard tunnel. 
  
    ```sh
    mkdir -p /mnt/data/split-vpn/wireguard/mullvad
    cd /mnt/data/split-vpn/wireguard/mullvad
    cp /mnt/data/split-vpn/vpn/vpn.conf.sample /mnt/data/split-vpn/wireguard/mullvad/vpn.conf
    vim wg0.conf
    [Press 'i' to start editing, right click -> paste, press 'ESC' to exit insert mode, type ':wq' to save and exit].
    ```
  
4. In your WireGuard config (wg0.conf), set PreUp, PostUp, and PreDown to point to the updown.sh script, and Table to a custom route table number that you will use in this script's vpn.conf. Here is an exmaple wg0.conf file:
  
    ```
    [Interface]
    PrivateKey = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    Address = 10.68.1.88/32,fc00:dddd:eeee:bb01::5:6666/128
    PreUp = sh /mnt/data/wireguard/updown.sh %i pre-up
    PostUp = sh /mnt/data/wireguard/updown.sh %i up
    PreDown = sh /mnt/data/wireguard/updown.sh %i down
    Table = 101

    [Peer]
    PublicKey = yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
    AllowedIPs = 0.0.0.0/1,128.0.0.0/1,::/1,8000::/1
    Endpoint = [2607:f7a0:d:4::a02f]:51820
    ```
  
    In the above config, make sure to:
      * Comment out or remove the `DNS` line. Use the DNS settings in your `vpn.conf` file instead if you want to force your clients to use a certain DNS server. 
      * Set AllowedIPs to `0.0.0.0/1,128.0.0.0/1,::/1,8000::/1` to allow all IPv4 and IPv6 traffic through the VPN. Do not use `0.0.0.0/0,::/0` because it will interfere with the blackhole routes and won't allow wireguard to start. If you prefer to use `0.0.0.0/0,::/0`, disable blackhole routes by setting `DISABLE_BLACKHOLE=1` in your `vpn.conf` file so wireguard can start successfully. 
      * Remove any extra PreUp/PostUp/PreDown/PostDown lines that could interfere with the VPN script. 
      * You can remove or comment out the PreUp line if you do not want VPN-forced clients to lose Internet access if WireGuard does not start correctly.

5. Edit the `vpn.conf` file with your desired settings. See the explanation of each setting [below](#configuration-variables). Make sure that:
  
   * The option `DNS_IPV4_IP` and/or `DNS_IPV6_IP` is set to the DNS server you want to force for your clients, or set them to empty if you do not want to force any DNS. 
   * The option `VPN_PROVIDER` is set to "external".
   * The option `VPN_ENDPOINT_IPV4` or `VPN_ENDPOINT_IPV6` is set to your WireGuard server's IP as defined in `wg0.conf`'s `Endpoint` variable.
   * The option `ROUTE_TABLE` is the same number as `Table` in your `wg0.conf` file.
   * The option `DEV` is set to "wg0" or your interface's name if different (i.e.: the name of your .conf file).
  
6. Run wg-quick to start wireguard with your configuration and test if the connection worked. 

    ```sh
    /mnt/data/wireguard/setup_wireguard.sh
    wg-quick up wg0.conf
    ```
  
    * You can skip the first line if you already setup the wireguard kernel module previously as instructed at [wireguard-kmod](https://github.com/tusc/wireguard-kmod).
    * Type `wg` to check your WireGuard connection and make sure you received a handshake. No handshake indicates something is wrong with your wireguard configuration. Double check your configuration's Private and Public key and other variables.
    * If you need to bring down the WireGuard tunnel, run `wg-quick down wg0.conf` in this folder.
    * Note that wg-quick up/down commands need to be run from this folder so the script can pick up the correct configuration file.
    
7. If the connection works, check each client to make sure they are on the VPN by doing the following.

    * Check if you are seeing the VPN IPs when you visit http://whatismyip.host/. You can also test from command line, by running the following commands from your clients (not the UDM/P). Make sure you are not seeing your real IP anywhere, either IPv4 or IPv6.
    
      ```sh
      curl -4 ifconfig.co
      curl -6 ifconfig.co
      ```
        
      If you are seeing your real IPv6 address above, make sure that you are forcing your client through IPv6 as well as IPv4, by forcing through interface, MAC address, or the IPv6 directly. If IPv6 is not supported by your VPN provider, the IPv6 check will time out and not return anything. You should never see your real IPv6 address. 

    * Check for DNS leaks with the Extended Test on https://www.dnsleaktest.com/. If you see a DNS leak, try redirecting DNS with the `DNS_IPV4_IP` and `DNS_IPV6_IP` options, or set `DNS_IPV6_IP="REJECT"` if your VPN provider does not support IPv6. 
    * Check for WebRTC leaks in your browser by visiting https://browserleaks.com/webrtc. If WebRTC is leaking your IPv6 IP, you need to disable WebRTC in your browser (if possible), or disable IPv6 completely by disabling it directly on your client or through the UDMP network settings for the client's VLAN.
    
8. If you want to block Internet access to forced clients if the wireguard tunnel is brought down via wg-quick, set `KILLSWITCH=1` and `REMOVE_KILLSWITCH_ON_EXIT=0` in the `vpn.conf` file. 
    
9. Now you can exit the UDM/P. If you would like to start the VPN client at boot, please read on to the next section. 

10. If your VPN provider doesn't support IPv6, it is recommended to disable IPv6 for that VLAN in the UDMP settings, or on the client, so that you don't encounter any delays. If you don't disable IPv6, clients on that network will try to communicate over IPv6 first and fail, then fallback to IPv4. This creates a delay that can be avoided if IPv6 is turned off completely for that network or client.
  
11. Note that the WireGuard protocol is practically stateless, so there is no way to know whether the connection stopped working except by checking that you didn't receive a handshake within 3 minutes or some higher interval. This means if you want to automatically bring down the split-vpn rules when WireGuard stops working and bring it back up when it starts working again, you need to write an external script to check the last handshake condition every few seconds and act on it (not covered here).

</details>

<details>
  <summary>Click here to see the instructions for wireguard-go (software implementation).</summary>

  * **Prerequisuite:** Make sure the wireguard-go container is installed as instructed at the [wireguard-go repo](https://github.com/boostchicken/udm-utilities/tree/master/wireguard-go). After this step, you should have the directory `/mnt/data/wireguard` and the run script `/mnt/data/on_boot.d/20-wireguard.sh` installed.
  * The wireguard-go container only supports a single interface - wg0. This means you cannot connect to multiple wireguard servers. If you want to use multiple servers with this script, then use the WireGuard kernel module instead as explained above. 
  * wireguard-go is a software implementation of WireGuard, and will have reduced performance compared to the kernel module. However, wireguard-go is not dependent on your UDM/P's kernel version, and will not break when your UDM/P updates.
  
1. SSH into the UDM/P (assuming it's on 192.168.1.254).

    ```sh
    ssh root@192.168.1.254
    ```
  
2. Download the scripts package and extract it to `/mnt/data/split-vpn/vpn`.

    ```sh
    cd /mnt/data
    mkdir /mnt/data/split-vpn && mkdir /mnt/data/split-vpn/vpn
    cd /mnt/data/split-vpn
    curl -L https://github.com/peacey/split-vpn/archive/main.zip | unzip - "*/vpn/*" -o -j -d vpn && chmod +x vpn/*.sh
    ```
    
3. Create a directory for your WireGuard configuration files under `/mnt/data/wireguard` if not created already, copy the sample vpn.conf from `/mnt/data/split-vpn/vpn/vpn.conf.sample`, and copy your WireGuard configuration file (wg0.conf) or create it. As an example below, we are creating the wg0.conf file that mullvad provides and pasting the contents into it. You can only use wg0.conf and not any other name because the wireguard-go container expects this configuration file. 
  
    ```sh
    mkdir -p /mnt/data/wireguard
    cd /mnt/data/wireguard
    cp /mnt/data/split-vpn/vpn/vpn.conf.sample /mnt/data/wireguard/vpn.conf
    vim wg0.conf
    [Press 'i' to start editing, right click -> paste, press 'ESC' to exit insert mode, type ':wq' to save and exit].
    ```
  
4. In your WireGuard config (wg0.conf), set Table to a custom route table number that you will use in this script's vpn.conf. Here is an exmaple wg0.conf file:
  
    ```
    [Interface]
    PrivateKey = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    Address = 10.68.1.88/32,fc00:dddd:eeee:bb01::5:6666/128
    Table = 101

    [Peer]
    PublicKey = yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
    AllowedIPs = 0.0.0.0/1,128.0.0.0/1,::/1,8000::/1
    Endpoint = [2607:f7a0:d:4::a02f]:51820
    ```
  
    In the above config, make sure to:
      * Comment out or remove the `DNS` line. Use the DNS settings in your `vpn.conf` file instead if you want to force your clients to use a certain DNS server. 
      * Set AllowedIPs to `0.0.0.0/1,128.0.0.0/1,::/1,8000::/1` to allow all IPv4 and IPv6 traffic through the VPN. Do not use `0.0.0.0/0,::/0` because it will interfere with the blackhole routes and won't allow wireguard to start. If you prefer to use `0.0.0.0/0,::/0`, disable blackhole routes by setting `DISABLE_BLACKHOLE=1` in your `vpn.conf` file so wireguard can start successfully. 
      * Remove any extra PreUp/PostUp/PreDown/PostDown lines that could interfere with the VPN script.
      * Do not use PreUp/PostUp/PreDown to call the split-vpn script (like in the wireguard kernel module case) because the container will not have access to the location of the script. Instead, we will call the script manually after bringing the interface up/down as instructed below. 

5. Edit the `vpn.conf` file with your desired settings. See the explanation of each setting [below](#configuration-variables). Make sure that:
  
   * The option `DNS_IPV4_IP` and/or `DNS_IPV6_IP` is set to the DNS server you want to force for your clients, or set them to empty if you do not want to force any DNS. 
   * The option `VPN_PROVIDER` is set to "external".
   * The option `VPN_ENDPOINT_IPV4` or `VPN_ENDPOINT_IPV6` is set to your WireGuard server's IP as defined in `wg0.conf`'s `Endpoint` variable.
   * The option `ROUTE_TABLE` is the same number as `Table` in your `wg0.conf` file.
   * The option `DEV` is set to "wg0".

6. Modify your run script that you installed with wireguard-go (located at `/mnt/data/on_boot.d/20-wireguard.sh`) to include the split-vpn script hooks by replacing it with the following code:
  
    ```sh
    #!/bin/sh
    CONTAINER=wireguard

    # Change to the directory with the wireguard configuration.
    cd /mnt/data/wireguard

    # Start the split-vpn pre-up hook.
    /mnt/data/split-vpn/vpn/updown.sh wg0 pre-up &> pre-up.log

    # Starts a wireguard container that is deleted after it is stopped.
    # All configs stored in /mnt/data/wireguard
    if podman container exists ${CONTAINER}; then
      podman start ${CONTAINER}
    else
      podman run -i -d --rm --net=host --name ${CONTAINER} --privileged \
        -v /mnt/data/wireguard:/etc/wireguard \
        -v /dev/net/tun:/dev/net/tun \
        -e LOG_LEVEL=info -e WG_COLOR_MODE=always \
        masipcat/wireguard-go:latest-arm64v8
    fi

    # Run the split-vpn up hook if wireguard starts successfully within 5 seconds.
    started=0
    for i in $(seq 1 5); do
            podman exec -it wireguard test -S "/var/run/wireguard/wg0.sock" &> /dev/null
            if [ $? = 0 ]; then
                    started=1
                    break
            fi
            sleep 1
    done
    if [ $started = 1 ]; then
            echo "wireguard-go started successfully." > wireguard.log
            /mnt/data/split-vpn/vpn/updown.sh wg0 up >> wireguard.log 2>&1
    else
            echo "Error: wireguard-go did not start up correctly within 5 seconds." > wireguard.log
    fi
    cat wireguard.log
    ```
  
    * Comment out the pre-up line at the beginning if you want your forced clients to be able to access the Internet if wireguard fails to start (i.e. commenting it out doesn't enable the iptables kill switch until after the VPN tunnel is brought up).
    * The above script will wait up to 5 seconds for the wireguard-go container to start before running the split-vpn up hook to set up the split-vpn rules. The split-vpn up hook will not be run if wireguard-go did not start up correctly. 

7. Run the 20-wireguard.sh script: `/mnt/data/on_boot.d/20-wireguard.sh`.
  
8. If wireguard-go started successfully, check that the connection worked by seeing if you received a handshake with the following command:
  
    ```sh
    podman exec -it wireguard wg
    ```
  
    * No handshake in the above output indicates something is wrong with your wireguard configuration. Double check your configuration's Private and Public key and other variables.
    * If you need to bring down the WireGuard tunnel and resume normal Internet access to your forced clients, run the following commands in this folder:
  
      ```sh
      cd /mnt/data/wireguard
      podman stop wireguard
      /mnt/data/split-vpn/vpn/updown.sh wg0 down
      ```
  
    * Note that split-vpn up/down commands need to be run from this folder so that split-vpn can pick up the correct configuration file.
    
9. If the connection works, check each client to make sure they are on the VPN by doing the following.

    * Check if you are seeing the VPN IPs when you visit http://whatismyip.host/. You can also test from command line, by running the following commands from your clients (not the UDM/P). Make sure you are not seeing your real IP anywhere, either IPv4 or IPv6.
    
      ```sh
      curl -4 ifconfig.co
      curl -6 ifconfig.co
      ```
        
      If you are seeing your real IPv6 address above, make sure that you are forcing your client through IPv6 as well as IPv4, by forcing through interface, MAC address, or the IPv6 directly. If IPv6 is not supported by your VPN provider, the IPv6 check will time out and not return anything. You should never see your real IPv6 address. 

    * Check for DNS leaks with the Extended Test on https://www.dnsleaktest.com/. If you see a DNS leak, try redirecting DNS with the `DNS_IPV4_IP` and `DNS_IPV6_IP` options, or set `DNS_IPV6_IP="REJECT"` if your VPN provider does not support IPv6. 
    * Check for WebRTC leaks in your browser by visiting https://browserleaks.com/webrtc. If WebRTC is leaking your IPv6 IP, you need to disable WebRTC in your browser (if possible), or disable IPv6 completely by disabling it directly on your client or through the UDMP network settings for the client's VLAN.
    
10. If you want to continue blocking Internet access to forced clients after the wireguard tunnel is brought down via the split-vpn down command, set `KILLSWITCH=1` and `REMOVE_KILLSWITCH_ON_EXIT=0` in the `vpn.conf` file. 
    
11. Now you can exit the UDM/P. If you would like to start the VPN client at boot, please read on to the next section. 

12. If your VPN provider doesn't support IPv6, it is recommended to disable IPv6 for that VLAN in the UDMP settings, or on the client, so that you don't encounter any delays. If you don't disable IPv6, clients on that network will try to communicate over IPv6 first and fail, then fallback to IPv4. This creates a delay that can be avoided if IPv6 is turned off completely for that network or client.
  
13. Note that the WireGuard protocol is practically stateless, so there is no way to know whether the connection stopped working except by checking that you didn't receive a handshake within 3 minutes or some higher interval. This means if you want to automatically bring down the split-vpn rules when WireGuard stops working and bring it back up when it starts working again, you need to write an external script to check the last handshake condition every few seconds and act on it (not covered here).
  
</details>

## How do I run this at boot?

You can use [UDM Utilities Boot Script](https://github.com/boostchicken/udm-utilities/tree/master/on-boot-script) to run the split-vpn script at boot. The boot script survives across firmware upgrades too. 
  
Set-up UDM Utilities Boot Script by following the instructions [here](https://github.com/boostchicken/udm-utilities/blob/master/on-boot-script/README.md) before following the instructions below.

<details>
  <summary>Click here to see the instructions for OpenVPN.</summary>
    
1. Create a new file under `/mnt/data/on_boot.d/99-run-vpn.sh` and fill it with the following. 

    ```sh
    #!/bin/sh
    # Load configuration and run openvpn
    cd /mnt/data/split-vpn/openvpn/nordvpn
    source ./vpn.conf
    /mnt/data/split-vpn/vpn/updown.sh ${DEV} pre-up &> pre-up.log
    nohup openvpn --config nordvpn.ovpn \
                  --route-noexec \
                  --up /mnt/data/split-vpn/vpn/updown.sh \
                  --down /mnt/data/split-vpn/vpn/updown.sh \
                  --dev-type tun --dev ${DEV} \
                  --script-security 2 \
                  --ping-restart 15 \
                  --mute-replay-warnings &> openvpn.log &
    ```

    * Remember to modify the `cd` line and the `--config` openvpn option to point to your config. 
    * Comment out the pre-up line if you want your forced clients to be able to access the Internet while the VPN is connecting (i.e. commenting it out doesn't enable the iptables kill switch until after OpenVPN connects).

2. Run `chmod +x /mnt/data/on_boot.d/99-run-vpn.sh` to give the script execute permissions. 
3. That's it. Now the VPN will start at every boot. 
4. Note that there is a short period between when the UDMP starts and when this script runs. This means there is a few seconds when the UDMP starts up when your forced clients **WILL** have access to your WAN and might leak their real IP, because the kill switch has not been activated yet. Read the queston *How can I block Internet access until after this script runs at boot?* in the [FAQ below](#faq) to see how to solve this problem and block Internet access until after this script runs. 
  
</details>

<details>
  <summary>Click here to see the instructions for WireGuard (kernel module).</summary>
    
1. Create a new file under `/mnt/data/on_boot.d/99-run-vpn.sh` and fill it with the following. 

    ```sh
    #!/bin/sh
  
    # Set up the wireguard kernel module and tools
    /mnt/data/wireguard/setup_wireguard.sh
  
    # Load configuration and run wireguard
    cd /mnt/data/split-vpn/wireguard/mullvad
    source ./vpn.conf
    /mnt/data/split-vpn/vpn/updown.sh ${DEV} pre-up &> pre-up.log
    wg-quick up ./${DEV}.conf &> wireguard.log
    cat wireguard.log
    ```

    * Comment out the pre-up line if you want your forced clients to be able to access the Internet if wireguard fails to start (i.e. commenting it out doesn't enable the iptables kill switch until after the VPN tunnel is brought up).
    * It is preferrable to separately run the pre-up hook like the above script instead of using wg-quick's PreUp hook in case wg-quick did not get set up correctly. wg-quick and the wireguard kernel module depends on the kernel version so might fail spontaneously if your UDM/P performs a software update.
    * You can remove the setup_wireguard.sh line if you have another boot script that sets up the kernel module before this run script. Just make sure that this script runs after the setup script by prefixing each script with a number to determine priority (e.g.: 90-setup-wireguard.sh and 99-run-vpn.sh).

2. Run `chmod +x /mnt/data/on_boot.d/99-run-vpn.sh` to give the script execute permissions. 
3. That's it. Now the VPN will start at every boot. 
4. **OPTIONAL**: Note that there is a short period between when the UDMP starts and when this script runs. This means there is a few seconds when the UDMP starts up when your forced clients **WILL** have access to your WAN and might leak their real IP, because the kill switch has not been activated yet. Read the queston *How can I block Internet access until after this script runs at boot?* in the [FAQ below](#faq) to see how to solve this problem and block Internet access until after this script runs. 
  
</details>

<details>
  <summary>Click here to see the instructions for wireguard-go.</summary>
    
1. The wireguard-go boot script should already be installed at `/mnt/data/on_boot.d/20-wireguard.sh` if you followed the instructions [above](#how-do-i-use-this), so no further setup is needed to make it run at boot.
  
2. **OPTIONAL**: Note that there is a short period between when the UDMP starts and when this script runs. This means there is a few seconds when the UDMP starts up when your forced clients **WILL** have access to your WAN and might leak their real IP, because the kill switch has not been activated yet. Read the queston *How can I block Internet access until after this script runs at boot?* in the [FAQ below](#faq) to see how to solve this problem and block Internet access until after this script runs.
  
</details>

## FAQ

<details>
  <summary>How can I block Internet access until after this script runs at boot?</summary>
    
  * If you want to ensure that there is no Internet access BEFORE this script runs at boot, you can add blackhole static routes in the Unifi Settings that will block all Internet access (incluing non-VPN Internet) until they are removed by this script. The blackhole routes will be removed when this script starts to restore Internet access only after the killswitch has been activated. If you want to do this for maximum protection at boot up, follow these instructions:

      1. Go to your Unifi Network Settings, and add the following static routes. If you're using the New Settings, this is under Advanced Features -> Advanced Gateway Settings -> Static Routes. For Old Settings, this is under Settings -> Routing and Firewall -> Static Routes. Add these routes which cover all IP ranges:

          * **Name:** VPN Blackhole. **Destination:** 0.0.0.0/1. **Static Route Type:** Black Hole. **Enabled.**
          * **Name:** VPN Blackhole. **Destination:** 128.0.0.0/1. **Static Route Type:** Black Hole. **Enabled.**
          * **Name:** VPN Blackhole. **Destination:** ::/1. **Static Route Type:** Black Hole. **Enabled.**
          * **Name:** VPN Blackhole. **Destination:** 8000::/1. **Static Route Type:** Black Hole. **Enabled.**

      2. In your vpn.conf, set the option `REMOVE_STARTUP_BLACKHOLES=1`. This is required or else the script will not delete the blackhole routes at startup, and you will not have Internet access on ANY client, not just the VPN-forced clients, until you delete the blackhole routes manually or disable them in the Unifi Settings.

      3. In your run script above, make sure you did NOT comment out the pre-up line. That is the line that removes the blackhole routes at startup.

      4. **Note that once you do this, you will lose Internet access for ALL clients until you run the VPN run script above**, or were running it before with the `REMOVE_STARTUP_BLACKHOLES=1` option. The split-vpn script stays running in the background to monitor if the the blackhole routes are added by the system again (which happens when your IP changes or when route settings are changed). The blackhole routes will be deleted immediately when they're added by the system.
  
</details>

<details>  
  <summary>Can I route clients to different VPN servers for OpenVPN?</summary>
  
  * Yes you can. Simply make a separate directory for each VPN server, and give them each a vpn.conf file with the clients you wish to force through them. Make sure the options `ROUTE_TABLE`, `MARK`, `PREFIX`, `PREF`, and `DEV` are unique for each `vpn.conf` file so the different VPN servers don't share the same tunnel device, route table, or fwmark. 
  
  * Afterwards, modify your run script like so (in this example, we are using Mullvad and NordVPN). Note that you need to cd into the correct configuration directory for each different VPN server before running the openvpn command so that the correct config file is used each time.
  
      ```sh
      #!/bin/sh

      # Load configuration for mullvad and run openvpn
      cd /mnt/data/split-vpn/openvpn/mullvad
      source ./vpn.conf
      /mnt/data/split-vpn/vpn/updown.sh ${DEV} pre-up &> pre-up.log
      nohup openvpn --config mullvad.conf \
                    --route-noexec \
                    --up /mnt/data/split-vpn/vpn/updown.sh \
                    --down /mnt/data/split-vpn/vpn/updown.sh \
                    --script-security 2 \
                    --dev-type tun --dev ${DEV} \
                    --ping-restart 15 \
                    --mute-replay-warnings &> openvpn.log &

      # Load configuration for nordvpn and run openvpn
      cd /mnt/data/split-vpn/openvpn/nordvpn
      source ./vpn.conf
      /mnt/data/split-vpn/vpn/updown.sh ${DEV} pre-up &> pre-up.log
      nohup openvpn --config nordvpn.ovpn \
                    --route-noexec \
                    --up /mnt/data/split-vpn/vpn/updown.sh \
                    --down /mnt/data/split-vpn/vpn/updown.sh \
                    --script-security 2 \
                    --dev-type tun --dev ${DEV} \
                    --ping-restart 15 \
                    --mute-replay-warnings &> openvpn.log &
    ```

</details>

<details>  
  <summary>Can I route clients to different VPN servers for WireGuard (kernel module)?</summary>
  
  * Yes you can. Simply make a separate directory for each VPN server, and give them each a vpn.conf file with the clients you wish to force through them. Make sure the options `ROUTE_TABLE`, `MARK`, `PREFIX`, `PREF`, and `DEV` are unique for each `vpn.conf` file so the different VPN servers don't share the same tunnel device, route table, or fwmark. 
  
  * Afterwards, modify your run script like so. Note that you need to cd into the correct configuration directory for each different VPN server before running the wg-quick command so that the correct config file is used each time.
  
    ```sh
    #!/bin/sh

    # Set up the wireguard kernel module and tools
    /mnt/data/wireguard/setup_wireguard.sh

    # Load configuration for mullvad and run wg-quick
    cd /mnt/data/split-vpn/wireguard/mullvad
    source ./vpn.conf
    /mnt/data/split-vpn/vpn/updown.sh ${DEV} pre-up &> pre-up.log
    wg-quick up ./${DEV}.conf &> wireguard.log

    # Load configuration for ExpressVPN and run wg-quick
    cd /mnt/data/split-vpn/wireguard/expressvpn
    source ./vpn.conf
    /mnt/data/split-vpn/vpn/updown.sh ${DEV} pre-up &> pre-up.log
    wg-quick up ./${DEV}.conf &> wireguard.log
    ```

</details>

<details>  
  <summary>Can I route clients to different VPN servers for wireguard-go?</summary>
  
  * No, you cannot. The wireguard-go container only supports a single interface named wg0. It cannot be used to connect an interface named something else, so only one interface is supported with this configuration. For multiple interfaces, you need to either use the kernel module or build your own docker container for wireguard-go that supports multiple interfaces (not covered here).
  
</details>

<details>
  <summary>Can I force/exempt domains to the VPN instead of just IPs?</summary>
  
  * Yes you can if you are using dnsmasq or pihole. Please see the [instructions here](ipsets/README.md) for how to set this up.
  
</details>

<details>
  <summary>I cannot access my WAN IP from a VPN-forced client (i.e. hairpin NAT does not work). What do I do?</summary>

  * In order for hairpin NAT to work, you need to exempt your WAN IPs from the VPN. You can do this in one of two ways:
  
    * If your WAN IP address is static and doesn't change, add your WAN's IPv4 address to `EXEMPT_DESTINATIONS_IPV4` and your IPv6 address to `EXEMPT_DESTINATIONS_IPV6`.
  
    * If your WAN IP changes often and you do not want to keep updating it in the script, exempt the Unifi provided IP sets that store your IPs by using the `EXEMPT_IPSETS` option. These IP sets are labelled `UBIOS_ADDRv4_<interface>`/`UBIOS_ADDRv6_<interface>` and are dynamically updated by Unifi when your WAN IP changes. For example, to exempt the WAN IP of eth8, use the following option. Note that for prefix delegation, the WAN IPv6 addresses are stored on the bridge interfaces, not the eth interfaces so make sure to add the bridge interface for IPv6 hairpin NAT. 
  
      ```sh
      EXEMPT_IPSETS="UBIOS_ADDRv4_eth8:dst UBIOS_ADDRv6_br0:dst"
      ```
  
</details>  

<details>
  <summary>How do I safely shutdown the VPN?</summary>
    
  * Run the following commands for your VPN type to bring down the VPN. Kill switch and iptables rules will only be removed if the option `REMOVE_KILLSWITCH_ON_EXIT` is set to 1.
  * If you set `REMOVE_KILLSWITCH_ON_EXIT=0` and want to recover Internet access for forced clients, please read the next question after you bring down the VPN.
  
    * **OpenVPN:** Send the openvpn process the TERM signal to bring it down.

        1. If you want to kill all openvpn instances.

            ```sh
            killall -TERM openvpn
            ```

        2. If you want to kill a specific openvpn instance using tun0.

            ```sh
            kill -TERM $(pgrep -f "openvpn.*tun0")
            ```
  
    * **WireGuard (kernel module):** Change to the directory of your vpn.conf configuration and run the wg-quick down command.
  
      ```sh
      cd /mnt/data/split-vpn/wireguard/mullvad
      wg-quick down ./wg0.conf
      ```
    * **wireguard-go:** Stop the container and run the split-vpn down command in the wireguard configuration directory.
    
      ```sh
      cd /mnt/data/wireguard
      podman stop wireguard
      /mnt/data/split-vpn/vpn/updown.sh wg0 down
      ```
  
</details>

<details>
  <summary>The VPN exited or crashed and now I can't access the Internet on my devices. What do I do?</summary>
  
  * When the VPN process crashes, there is no cleanup done for the iptables rules and the kill switch is still active (if the kill switch is enabled). This is also the case for a clean exit when you set the option `REMOVE_KILLSWITCH_ON_EXIT=0`. This is a safety feature so that there are no leaks if the VPN crashes. To recover the connection, do the following:
  
    * If you don't want to delete the kill switch and leak your real IP, re-run the run script or command to bring the VPN back up again.

    * If you want to delete the kill switch so your forced clients can access your normal Internet again, run the following command (replace tun0 with the device you defined in the config file) after changing to the directory with the vpn.conf file. This command applies to OpenVPN and WireGuard.
    
        ```sh
        cd /mnt/data/split-vpn/openvpn/nordvpn
        /mnt/data/split-vpn/vpn/updown.sh tun0 force-down
        ```
        
    * If you added blackhole routes and deleted the kill switch in the previous step, make sure to disable the blackhole routes in the Unifi Settings or you might suddenly lose Internet access when the blackhole routes are re-added by the system.
      
</details>

<details>
  <summary>How do I enable or disable the kill switch and what does it do?</summary>
  
  * The kill switch disables Internet access for VPN-forced clients if the VPN crashes or exits. This is good for when you do not want to leak your real IP if the VPN crashes or exits prematurely. Follow the instructions below to enable or disable the kill switch.
  
    1. To enable the kill switch, set `KILLSWITCH=1` in the `vpn.conf` file. If you want the kill switch to remain even when you bring down the VPN cleanly (not just when it crashes), then set `REMOVE_KILLSWITCH_ON_EXIT=0` as well. 

    2. To disable the kill switch, set `KILLSWITCH=0` and `REMOVE_KILLSWITCH_ON_EXIT=1`. Note that there will be nothing preventing your VPN-forced clients from leaking their real IP if you disable the kill switch. 

    3. If you previously had the kill switch enabled and want to disable it after a crash or exit to recover Internet access, read the previous question. 
  
</details>
  
<details>
  <summary>How do I enable or disable the VPN blackhole routes and what do they do?</summary>
  
  * The VPN blackhole routes are added to the custom route table before the VPN routes are added. The blackhole routes prevent Internet access on VPN-forced clients if the VPN started successfully but the VPN routes were not added because of some problem. Note this is not the same as the Unifi system-wide blackhole routes to prevent Internet access on startup before the VPN script runs (outlined in the boot section above).
  
    1. To enable the VPN blackhole routes, make sure `DISABLE_BLACKHOLE=0` or the variable is not set in your vpn.conf (default).
  
    2. To disable the VPN blackhole routes, set `DISABLE_BLACKHOLE=1` in your vpn.conf. This is not recommended unless you do not care about your real IP leaking. 
  
</details>

<details>
  <summary>How do I check port forwarding on the VPN side is working?</summary>
  
  * Use a port checking tool (like https://websistent.com/tools/open-port-check-tool/) and enter your VPN IP and VPN port number to test. Check both IPv6 and IPv4 if using both. 
  * Alternatively, you can run the following command on your client which tells your IP and if the port is open. Make sure you are not seeing your real IP here and that the status for the port is reachable. Replace 21674 with your VPN port number. 
  
    ```sh
     curl -4 https://am.i.mullvad.net/port/21674
     curl -6 https://am.i.mullvad.net/port/21674
     ```
     
</details>

<details>
  <summary>Why I am seeing my real IPv6 address when checking my IP on the Internet?</summary>
  
  * You shouldn't be seeing your real IPv6 address anywhere if you forced your clients over IPv6, even if your VPN doesn't support IPv6. Make sure that you are forcing your client through IPv6 as well as IPv4, by forcing through interface (`FORCED_SOURCE_INTERFACE`), MAC address (`FORCED_SOURCE_MAC`), or the IPv6 directly (`FORCED_SOURCE_IPV6`). 
  
  * If IPv6 is not supported by your VPN provider, IPv6 traffic should time out or be refused. For additional security if your VPN provider doesn't support IPv6, it is recommended to set the `DNS_IPV6_IP` option to "REJECT", or disable IPv6 for that network in the UDMP settings, so that IPv6 DNS leaks do not occur.
     
</details>

<details>
  <summary>Why am I seeing my real IPv6 address when doing a WebRTC test?</summary>
  
  * WebRTC is an audio/video protocol that allows browsers to get your IPs via JavaScript. WebRTC cannot be completely disabled at the network level because some browsers check the network interface directly to see what IP to return. Since IPv6 has global IPs directly assigned to the network interface, your non-VPN global IPv6 can be directly seen by the browser and leaked to WebRTC JavaScript calls. To solve this, you can do one of the following.
  
    * Disable WebRTC on your browser (not all browsers allow you to). Use a browser like Firefox that allows you to disable WebRTC.
    * Disable JavaScript completely on your browser (which will break most sites).
    * Disable IPv6 completely either directly on the client (if you can), or by using the UDMP's network settings to turn off IPv6 for the client's VLAN.
  
</details>

<details>
  <summary>My VPN provider doesn't support IPv6. Why do my forced clients have a delay in communicating to the Internet?</summary>
  
  * If your VPN provider doesn't support IPv6 but you have IPv6 enabled on the network, clients will attempt to communicate over IPv6 first then fallback to IPv4 when the connection fails, since IPv6 is not supported on the VPN. 
  
  * To avoid this delay, it is recommended to disable IPv6 for that network/VLAN in the UDMP settings, or on the client directly. This ensures that the clients only use IPv4 and don't have to wait for IPv6 to time out first.
     
</details>

<details>
  <summary>Does the VPN still work when my IP changes because of lease renewals or other disconnect reasons?</summary>
    
  * **For OpenVPN:** Yes, as long as you add the `--ping-restart X` option to the openvpn command line when you run it. This ensures that if there is a network disconnect for any reason, the OpenVPN client will restart and try to re-configure itself after X seconds until it connects again. The killswitch will still be active during the restart to block non-VPN traffic as long as you set `REMOVE_KILLSWITCH_ON_EXIT=0` in the config.
  
  * **For WireGuard:** The WireGuard protocol is practically stateless, so IP changes will not affect the connection. 
    
</details>

<details>
  <summary>What does this really do to my UDMP?</summary>
  
  * This script only does the following.
  
    1. Adds custom iptable chains and rules to the mangle, nat, and filter tables. You can see them with the following commands (assuming you set `PREFIX=VPN_`).

        ```sh
        iptables -t mangle -S | grep VPN
        iptables -t nat -S | grep VPN
        iptables -t filter -S | grep VPN
        ip6tables -t mangle -S | grep VPN
        ip6tables -t nat -S | grep VPN
        ip6tables -t filter -S | grep VPN
        ```

    2. Adds VPN routes to custom routing tables that can be seen with the following command (assuming you set `ROUTE_TABLE=101`).

        ```sh
        ip route show table 101
        ip -6 route show table 101
        ```

    2. Adds policy-based routes to redirect marked traffic to the custom tables. You can see them with the following command (look for the fwmark you defined in your config or 0x9 if using default).

        ```sh
        ip rule
        ip -6 rule
        ```

    4. Stays running in the background to monitor the policy-based routes every second for any deletions caused by the UDMP operating system, and re-adds them if deleted. The UDMP removes the custom policy-based routes when the WAN IP changes. You can see the script running with:

        ```sh
        ps | grep updown.sh
        ```

    5. Writes logs to `openvpn.log` (if using openvpn) and `rule-watcher.log` in each VPN server's directory. Logs are overwritten at every run.
      
</details>

<details>
  <summary>Something went wrong. How can I debug?</summary>
  
  1. For OpenVPN, first check the openvpn.log file in the VPN server's directory for any errors. 
  2. For WireGuard, check wireguard.log after you run your run script or check the output of wg-quick up. For wireguard-go, check the output when you run your run script. Make sure you received a handshake in WireGuard or the connection will not work. If you did not receive a handshake, double check your configuration's Private and Public key and other variables.
  2. Check that the iptable rules, policy-based routes, and custom table routes agree with your configuration. See the previous question for how to look this up.
  3. If you want to see which line the scripts failed on, open the `updown.sh` and `add-vpn-iptables-rules.sh` scripts and replace the `set -e` line at the top with `set -xe` then rerun the VPN. The `-x` flag tells the shell to print every line before it executes it. 
  4. Post a bug report if you encounter any reproducible issues. 
  
</details>

## Configuration variables

<details>
  <summary>Settings are modified in vpn.conf. Multiple entries can be entered for each setting by separating the entries with spaces. Click here to see all the settings.</summary>
  
  <details>
    <summary>FORCED_SOURCE_INTERFACE</summary>
      Force all traffic coming from a source interface through the VPN. 
      Default LAN is br0, and other LANs are brX, where X = VLAN number.

      Format: [INTERFACE NAME]
      Example: FORCED_SOURCE_INTERFACE="br6 br8"

  </details>
  
  <details>
    <summary>FORCED_SOURCE_IPV4</summary>
      Force all traffic coming from a source IPv4 through the VPN. 
      IP can be entered in CIDR format to cover a whole subnet. 
  
      Format: [IP/nn]
      Example: FORCED_SOURCE_IPV4="192.168.1.1/32 192.168.3.0/24"

  </details>
  
  <details>
    <summary>FORCED_SOURCE_IPV6</summary>
      Force all traffic coming from a source IPv6 through the VPN. 
      IP can be entered in CIDR format to cover a whole subnet. 
  
      Format: [IP/nn]
      Example: FORCED_SOURCE_IPV6="fd00::2/128 2001:1111:2222:3333::/56"

  </details>
  
  <details>
    <summary>FORCED_SOURCE_MAC</summary>
      Force all traffic coming from a source MAC through the VPN.
  
      Format: [MAC]
      Example: FORCED_SOURCE_MAC="00:aa:bb:cc:dd:ee 30:08:d7:aa:bb:cc"

  </details>
  
  <details>
    <summary>FORCED_DESTINATIONS_IPV4</summary>
      Force IPv4 destinations to the VPN for all VPN-forced clients.
  
      Format: [IP/nn]
      Example: FORCED_DESTINATIONS_IPV4="1.1.1.1"

  </details>

  <details>
    <summary>FORCED_DESTINATIONS_IPV6</summary>
      Force IPv6 destinations to the VPN for all VPN-forced clients.
  
      Format: [IP/nn]
      Example: FORCED_DESTINATIONS_IPV6="2001:1111:2222:3333::2"

  </details>
  
  <details>
    <summary>FORCED_LOCAL_INTERFACE</summary>
      Force local UDM traffic going out of these interfaces to go through the VPN instead, for both IPv4 and IPv6 traffic. This does not include routed traffic, only local traffic generated by the UDM. 
    
      For UDM-Pro, set to "eth8" for WAN1/Ethernet port, or "eth9" for WAN2/SFP+ port, or "eth8 eth9" for both. 
      For UDM Base, set to "eth1" for the WAN port.
      Format: [INTERFACE NAME]
      Example: FORCED_LOCAL_INTERFACE="eth8 eth9"

  </details>
  
  <details>
    <summary>EXEMPT_SOURCE_IPV4</summary>
      Exempt IPv4 sources from the VPN. This allows you to create exceptions to the force rules above. For example, if you forced a whole interface with FORCED_SOURCE_INTERFACE, you can selectively choose clients from that VLAN to exclude.
  
      Format: [IP/nn]
      Example: EXEMPT_SOURCE_IPV4="192.168.1.2/32 192.168.3.8/32"

  </details>
  
  <details>
    <summary>EXEMPT_SOURCE_IPV6</summary>
      Exempt IPv6 sources from the VPN. This allows you to create exceptions to the force rules above. 
  
      Format: [IP/nn]
      Example: EXEMPT_SOURCE_IPV6="2001:1111:2222:3333::2 2001:1111:2222:3333::10"

  </details>
  
  <details>
    <summary>EXEMPT_SOURCE_MAC</summary>
      Exempt MAC sources from the VPN. This allows you to create exceptions to the force rules above. 
  
      Format: [MAC]
      Example: EXEMPT_SOURCE_MAC="00:aa:bb:cc:dd:ee 30:08:d7:aa:bb:cc"

  </details>
   
  <details>
    <summary>EXEMPT_SOURCE_IPV4_PORT</summary>
      Exempt an IPv4:Port source from the VPN. This allows you to create exceptions on a port basis, so you can selectively choose which services on a client to tunnel through the VPN and which to tunnel through the default LAN/WAN. For example, you can tunnel all traffic through the VPN for some client, but have port 22 still be accessible over the LAN/WAN so you can SSH to it normally. 
  
      A single entry can have up to 15 multiple ports by separating the ports with commas. 
      Ranges of ports can be defined with a colon like 5000:6000, and take up two ports in the entry. 
      Protocal can be tcp, udp or both. 
      Format: [tcp/udp/both]-[IP Source]-[port1,port2:port3,port4,...]
      Example: EXEMPT_SOURCE_IPV4_PORT="tcp-192.168.1.1-22,32400,80:90,443 both-192.168.1.3-53"

  </details>
  
  <details>
    <summary>EXEMPT_SOURCE_IPV6_PORT</summary>
      Exempt an IPv6:Port source from the VPN. This allows you to create exceptions on a port basis, so you can selectively choose which services on a client to tunnel through the VPN and which to tunnel through the default LAN/WAN. 
 
      A single entry can have up to 15 multiple ports by separating the ports with commas. 
      Ranges of ports can be defined with a colon like 5000:6000, and take up two ports in the entry. 
      Protocal can be tcp, udp or both. 
      Format: [tcp/udp/both]-[IP Source]-[port1,port2:port3,port4,...]
      Example: EXEMPT_SOURCE_IPV6_PORT="tcp-fd00::69-22,32400,80:90,443 both-fd00::2-53"

  </details> 
  
  <details>
    <summary>EXEMPT_SOURCE_MAC_PORT</summary>
      Exempt a MAC:Port source from the VPN. This allows you to create exceptions on a port basis, so you can selectively choose which services on a client to tunnel through the VPN and which to tunnel through the default LAN/WAN. 
 
      A single entry can have up to 15 multiple ports by separating the ports with commas. 
      Ranges of ports can be defined with a colon like 5000:6000, and take up two ports in the entry. 
      Protocal can be tcp, udp or both. 
      Format: [tcp/udp/both]-[MAC Source]-[port1,port2:port3,port4,...]
      Example: EXEMPT_SOURCE_MAC_PORT="both-30:08:d7:aa:bb:cc-22,32400,80:90,443"

  </details>
  
  <details>
    <summary>EXEMPT_DESTINATIONS_IPV4</summary>
      Exempt IPv4 destinations from the VPN for all VPN-forced clients.
  
      Format: [IP/nn]
      Example: EXEMPT_DESTINATIONS_IPV4="192.168.1.0/24 10.0.5.3/32"

  </details>

  <details>
    <summary>EXEMPT_DESTINATIONS_IPV6</summary>
      Exempt IPv6 destinations from the VPN for all VPN-forced clients.
      Format: [IP/nn]
      Example: EXEMPT_DESTINATIONS_IPV6="fd62:1200:1300:1400::2/32 2001:1111:2222:3333::/56"

  </details>
  
  <details>
    <summary>FORCED_IPSETS</summary>
      Force these IP sets through the VPN. IP sets need to be created before this script is run or the script will error. IP sets can be updated externally and will be matched dynamically. Each IP set entry consists of the IP set name and whether to match on source or destination for each field in the IP set. 
    
      Note: These IP sets will be forced for every VPN-forced client. If you want to force different IP sets for different clients, use `CUSTOM_FORCED_RULES_IPV4` and  `CUSTOM_FORCED_RULES_IPV6` below.
    
      src/dst needs to be specified for each IP set field.
      Format: Format: [IPSet Name]:[src/dst,src/dst,...]
      Example: FORCED_IPSETS="VPN_FORCED:dst IPSET_NAME:src,dst"

  </details>
  
  <details>
    <summary>EXEMPT_IPSETS</summary>
      Exempt these IP sets from the VPN. IP sets need to be created before this script is run or the script will error. IP sets can be updated externally and will be matched dynamically. Each IP set entry consists of the IP set name and whether to match on source or destination for each field in the IP set. 
      You can also use this option to allow NAT hairpin to work while on the VPN by adding the Unifi-provided IP sets for the interface to this variable. For example, for eth8 WAN, IP sets are UBIOS_ADDRv4_eth8 and UBIOS_ADDRv6_eth8. Note that for IPv6 prefix delegation, IPv6 WAN addresses are stored on the bridge interfaces, not eth interfaces. 
    
      Note: These IP sets will be exempt for every VPN-forced client. If you want to exempt different IP sets for different clients, use `CUSTOM_EXEMPT_RULES_IPV4` and  `CUSTOM_EXEMPT_RULES_IPV6` below.
    
      src/dst needs to be specified for each IP set field.
      Format: Format: [IPSet Name]:[src/dst,src/dst,...]
      Example: EXEMPT_IPSETS="VPN_EXEMPT:dst IPSET_NAME:src,dst"
               EXEMPT_IPSETS="UBIOS_ADDRv4_eth8:dst UBIOS_ADDRv6_br0:dst"

  </details>
  
  <details>
    <summary>CUSTOM_FORCED_RULES_IPV4</summary>
      Custom IPv4 rules that will be forced to the VPN. The format of these rules is the matching portion of the iptables command, without the table, chain, and jump target. Multiple rules can be added on separate lines. These rules are added to the mangle table and the PREROUTING chain. 

      Opening and closing quotation marks must not be removed if using multiple lines.
      Format: Format: [Matching portion of iptables command]
      Example: 
        CUSTOM_FORCED_RULES_IPV4="
            -s 192.168.1.6
            -p tcp -s 192.168.1.10 --dport 443
            -m set --match-set VPN_FORCED dst -i br6
        "

  </details>
  
  <details>
    <summary>CUSTOM_FORCED_RULES_IPV6</summary>
      Custom IPv6 rules that will be forced to the VPN. The format of these rules is the matching portion of the iptables command, without the table, chain, and jump target. Multiple rules can be added on separate lines. These rules are added to the mangle table and the PREROUTING chain. 
      
      Opening and closing quotation marks must not be removed if using multiple lines.
      Format: Format: [Matching portion of iptables command]
      Example: 
        CUSTOM_FORCED_RULES_IPV6="
            -s fd62:1200:1300:1400::2/32
            -p tcp -s fd62:1200:1300:1400::10 --dport 443
            -m set --match-set VPN_FORCED dst -i br6
        "

  </details>
  
  <details>
    <summary>CUSTOM_EXEMPT_RULES_IPV4</summary>
      Custom IPv4 rules that will be exempt from the VPN. The format of these rules is the matching portion of the iptables command, without the table, chain, and jump target. Multiple rules can be added on separate lines. These rules are added to the mangle table and the PREROUTING chain. 

      Opening and closing quotation marks must not be removed if using multiple lines.
      Format: Format: [Matching portion of iptables command]
      Example: 
        CUSTOM_EXEMPT_RULES_IPV4="
            -s 192.168.1.6
            -p tcp -s 192.168.1.10 --dport 443
            -m set --match-set VPN_EXEMPT dst -i br6
        "

  </details>
  
  <details>
    <summary>CUSTOM_EXEMPT_RULES_IPV6</summary>
      Custom IPv6 rules that will be exempt from the VPN. The format of these rules is the matching portion of the iptables command, without the table, chain, and jump target. Multiple rules can be added on separate lines. These rules are added to the mangle table and the PREROUTING chain. 
      
      Opening and closing quotation marks must not be removed if using multiple lines.
      Format: Format: [Matching portion of iptables command]
      Example: 
        CUSTOM_EXEMPT_RULES_IPV6="
            -s fd62:1200:1300:1400::2/32
            -p tcp -s fd62:1200:1300:1400::10 --dport 443
            -m set --match-set VPN_EXEMPT dst -i br6
        "

  </details>
  
  <details>
    <summary>PORT_FORWARDS_IPV4</summary>
      Forward ports on the VPN side to a local IPv4:port. Not all VPN providers support port forwards. The ports are usually given to you on the provider's portal.
  
      Only one port per entry. Protocal can be tcp, udp or both. 
      Format: [tcp/udp/both]-[VPN Port]-[Forward IP]-[Forward Port]
      Example: PORT_FORWARDS_IPV4="tcp-21674-192.168.1.1-50001 tcp-31683-192.168.1.1-22"

  </details>
  
  <details>
    <summary>PORT_FORWARDS_IPV6</summary>
      Forward ports on the VPN side to a local IPv6:port. Not all VPN providers support port forwards. The ports are usually given to you on the provider's portal.
  
      Only one port per entry. Protocal can be tcp, udp or both. 
      Format: [tcp/udp/both]-[VPN Port]-[Forward IP]-[Forward Port]
      Example: PORT_FORWARDS_IPV6="tcp-21674-2001:aaa:bbbb:2acc::69-50001 tcp-31456-2001:aaa:bbbb:2acc::70-443"

  </details>
  
  <details>
    <summary>DNS_IPV4_IP, DNS_IPV4_PORT</summary>
      Redirect DNS IPv4 traffic of VPN-forced clients to this IP and port.
      If set to "DHCP", the DNS will try to be obtained from the DHCP options that OpenVPN sends. DHCP option is only supported on OpenVPN.
      If set to "REJECT", DNS requests over IPv6 will be blocked instead. 
      Note that many VPN providers redirect all DNS traffic to their servers, so redirection to other IPs might not work on all providers.
      DNS redirects to a local address, or rejecting DNS traffic works for all providers.
      Make sure to set DNS_IPV4_INTERFACE if redirecting to a local DNS address. 
  
      Format: [IP] or "DHCP" or "REJECT"
      Example: DNS_IPV4_IP="1.1.1.1"
      Example: DNS_IPV4_IP="DHCP"
      Example: DNS_IPV4_IP="REJECT"
      Example: DNS_IPV4_PORT=53

  </details>
  
  <details>
    <summary>DNS_IPV4_INTERFACE</summary>
      Set this to the interface (brX) the IPv4 DNS is on if it is a local IP. Leave blank for non-local DNS. 
      Local DNS redirects will not work without specifying the interface.
  
      Format: [brX]
      Example: DNS_IPV4_INTERFACE="br0"

  </details>
  
  <details>
    <summary>DNS_IPV6_IP, DNS_IPV6_PORT</summary>
      Redirect DNS IPv6 traffic of VPN-forced clients to this IP and port. 
      If set to "REJECT", DNS requests over IPv6 will be blocked instead. The REJECT option is recommended to be enabled for VPN providers that don't support IPv6, to eliminate any IPv6 DNS leaks.
      Note that many VPN providers redirect all DNS traffic to their servers, so redirection to other IPs might not work on all providers.
      DNS redirects to a local address, or rejecting DNS traffic works for all providers.
      Make sure to set DNS_IPV6_INTERFACE if redirecting to a local DNS address. 
  
      Format: [IP] or "REJECT"
      Example: DNS_IPV6_IP="2606:4700:4700::64"
      Example: DNS_IPV6_IP="REJECT"
      Example: DNS_IPV6_PORT=53

  </details>
  
  <details>
    <summary>DNS_IPV6_INTERFACE</summary>
      Set this to the interface (brX) the IPv6 DNS is on if it is a local IP. Leave blank for non-local DNS. 
      Local DNS redirects will not work without specifying the interface.
  
      Format: [brX]
      Example: DNS_IPV6_INTERFACE="br0"

  </details>
  
  <details>
    <summary>KILLSWITCH</summary>
      Enable killswitch which adds an iptables rule to reject VPN-destined traffic that doesn't go out of the VPN. 
  
      Format: 0 or 1
      Example: KILLSWITCH=1

  </details>
  
  <details>
    <summary>REMOVE_KILLSWITCH_ON_EXIT</summary>
      Remove the killswitch on exit. 
      It is recommended to set this to 0 so that the killswitch is not removed in case the openvpn client crashes, disconnects, or restarts. 
      Setting this to 1 will remove the killswitch when the openvpn client restarts, which means clients might be able to communicate with your default WAN and leak your real IP while the openvpn client is restarting. 
  
      Format: 0 or 1
      Example: REMOVE_KILLSWITCH_ON_EXIT=0

  </details>
  
  <details>
    <summary>REMOVE_STARTUP_BLACKHOLES</summary>
    Enable this if you added blackhole routes in the Unifi Settings to prevent Internet access at system startup before the VPN script runs. 
    This option removes the blackhole routes to restore Internet access after the killswitch has been enabled.               
    If you do not set this to 1, openvpn will not be able to connect at startup, and your Internet access will never be enabled until you manually remove the blackhole routes. 
    Set this to 0 only if you did not add any blackhole routes in Step 6 of the boot script instructions above.                         
  
      Format: 0 or 1
      Example: REMOVE_STARTUP_BLACKHOLES=1

  </details>
  
  <details>
    <summary>DISABLE_BLACKHOLE</summary>
    Set this to 1 to disable the VPN blackhole routes that are added (the blackhole routes help prevent VPN-forced clients from accessing the Internet if the VPN routes are not added because an error). 
    
      Format: 0 (default) or 1
      Example: DISABLE_BLACKHOLE=0

  </details>
  
  <details>
    <summary>VPN_PROVIDER</summary>
     The VPN provider you are using with this script.
    
      Format: "openvpn" for OpenVPN (default), or "external" for wireguard or other external providers.
      Example: VPN_PROVIDER="openvpn"

  </details>
    
  <details>
    <summary>VPN_ENDPOINT_IPV4</summary>
    If using "external" for VPN_PROVIDER, set this to the VPN endpoint's IPv4 address so that the gateway route can be automatically added for the VPN endpoint. OpenVPN automatically passes the VPN endpoint IP to the script and will override this value.
    
      Format: [IP]
      Example: VPN_ENDPOINT_IPV4="2.2.2.2"

  </details>
  
  <details>
    <summary>VPN_ENDPOINT_IPV6</summary>
    If using "external" for VPN_PROVIDER, set this to the VPN endpoint's IPv6 address so that the gateway route can be automatically added for the VPN endpoint. OpenVPN automatically passes the VPN endpoint IP to the script and will override this value.
    
      Format: [IP]
      Example: VPN_ENDPOINT_IPV6="2606:43:ee::23"

  </details>
  
  <details>
    <summary>GATEWAY_TABLE</summary>
    Set this to the route table that contains the gateway route, or "auto". "201" if you're using Ethernet, "202" for SFP+, and "203" for U-LTE. Default is "auto" which works with WAN failover and automatically changes the endpoint via gateway route when the WAN or gateway routes changes.
    
      Format: [Route Table Number] or "auto" (default)
      Example: GATEWAY_TABLE="auto"
               GATEWAY_TABLE="201"

  </details>
  
  <details>
    <summary>WATCHER_TIMER</summary>
    Set this to the timer to use for the rule watcher (in seconds). The script will wake up every N seconds to re-add rules if they're deleted by the system, or change gateway routes if they changed. Default is 1 second. 
    
      Format: [Seconds]
      Example: WATCHER_TIMER=1

  </details>
  
  <details>
    <summary>BYPASS_MASQUERADE_IPV4</summary>
    Bypass masquerade (SNAT) for these IPv4s. This option should only be used if your VPN server is setup to know how to route the subnet you do not want to masquerade (e.g.: the "iroute" option in OpenVPN).
    Set this option to ALL to disable masquerading completely.
    
      Format: [IP/nn] or "ALL"
      Example: BYPASS_MASQUERADE_IPV4="10.100.1.0/24"

  </details>
  
  <details>
    <summary>BYPASS_MASQUERADE_IPV6</summary>
    Bypass masquerade (SNAT) for these IPv6s. This option should only be used if your VPN server is setup to know how to route the subnet you do not want to masquerade (e.g.: the "iroute" option in OpenVPN).
    Set this option to ALL to disable masquerading completely.
    
      Format: [IP/nn] or "ALL"
      Example: BYPASS_MASQUERADE_IPV6="fd64::/64"

  </details>
  
  <details>
    <summary>ROUTE_TABLE</summary>
      The custom route table number. 
      If you are running multiple openvpn clients, this needs to be unique for each client.
  
      Format: [Number]
      Example: ROUTE_TABLE=101

  </details>
  
  <details>
    <summary>MARK</summary>
      The firewall mark that will be used to mark the packets destined to the VPN. 
      If you are running multiple openvpn clients, this needs to be unique for each client.
  
      Format: [Hex number]
      Example: MARK=0x9

  </details>
  
  <details>
    <summary>PREFIX</summary>
      The prefix that will be used when adding custom iptables chains. 
      If you are running multiple openvpn clients, this needs to be unique for each client. 

      Format: [Prefix]
      Example: PREFIX=VPN_

  </details>
  
  <details>
    <summary>PREF</summary>
      The preference that will be used when adding the policy-based routing rule.
      It should preferably be less than the UDM rules seen when running "ip rule".

      Format: [Number]
      Example: PREF=99

  </details>
  
  <details>
    <summary>DEV</summary>
    The name of the VPN tunnel device to use for openvpn. 
    If you are running multiple openvpn clients, this needs to be unique for each client. 
    This variable needs to be passed to openvpn via the --dev option or openvpn will default to tun0.

      Format: [tunX]
      Example: DEV=tun0

  </details>
  
</details>
