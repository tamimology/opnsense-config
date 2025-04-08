# Vital Settings in OPNSense

#### **NOTE:** *WHEN WRITING THIS GUIDE, IT WAS BASED ON [OPNSENSE v24.7.6](https://opnsense.org/download/)*


# Table of Contents

***This guide presents important plugins, packages, settings, and configuration for OPNSense, it might be updated based on my findings***


1. <a href="#new-user-as-root">New user as ROOT</a>
2. <a href="#kea-dhcp">Kea DHCP</a>
3. <a href="#dyn-dns">Dyn-DNS</a>
4. <a href="#adguard-home">Adguard Home</a>
5. <a href="#geoip-blocking">GeoIP Blocking</a>
6. <a href="#clamav">ClamAV</a>
6. <a href="#spamhaus-lists">Spamhaus Lists</a>


### New User as ROOT

Once OPNSense is installed, the default user is `root` and the default password is `opnsense`. This user has ROOT power, it is advisable to disable this user and create another user with the same power and privileges.

To do so, In OPNSense, navigate to *System-->Access-->Users*, then click on the *+* button on the right end to add a new user, and fill in the following:

- Make sure _Disabled_ is not ticked
- *Username* : enter the desired username here, i.e. _myself_
- *Password* : enter the desired password here, twice
- *Login shell* : set it to _/bin/sh_ to allow this user SSH access, or keep it as _/usr/sbin/nologin_ to disable SSH access
- *Group Memberships* : select the _admins_ from the _*Not Member Of*_ list and click on the arrow heading towards the right list of *_Member Of*_ and make sure it is added there
- Keep everything as defaults

Once done, click on _Save and go back_

![user-add](/screenshots/user/user-add.png)


Now, logoff the _root_ user by heading to _Lobby-->Logout_ and then log in again using the newly created user above.

We need to disable the _root_ user, by navigating to *System-->Access-->Users*, click on the _PEN_ icon next to the _root_ user, and from that screen check the _Disabled_ option. Click on _Save and go back_

This user has SSH access (if it was enabled above in the _Login shell_ option), but it is not a _root_ privileged SSH. To grant this, do the following:
Navigate to _System-->Settings-->Administration_ and scroll down till you reach to the _*Secure Shell*_ section, and fill in the following:

- *Secure Shell Server* : check the box next to that says _Enable Secure Shell_
- *Login Group* : _wheel,admins_
_ *Root Login* : (OPTIONAL) check the box next to that says _Permit root user login_. This is just in case the _root_ user was enabled at a stage and requires SSH access
_ *Authentication Method* : check the box next to that says _Permit password login_
- *SSH port* : it is recommended to change this port from the default _22_ to anything else for more security
- *Listen Interfaces* : LAN

![user-ssh](/screenshots/user/user-ssh.png)

Now scroll to the end till you reach the _*Authentication*_ section, and fill as follows:
- *Server* : Nothing selected
- *Sudo* : Ask password
           wheel, admins 

![user-auth](/screenshots/user/user-auth.png)


Once done, click on _Save_

When you need to SSH with the root previlages, simply execute `sudo su -` and enter the password when prompted.


## Kea DHCP

OPNSense is set with the default _ISC DHCP_ which is obsolete now and is replaced by _Kea DHCP_. To enable the new one, first, we need to disable _ISC_ and then enable and configure _Kea_. To do so, navigate to *Services-->ISC DHCPv4-->[LAN]*

On that page, simply uncheck the _Enable DHCP server on the LAN interface_

Once done, click on _Save_

![user-auth](/screenshots/kea/isc-dhcpv4.png)


Now, navigate to *Services-->Kea DHCP [new]-->Kea DHCPv4*, and fill in the following:

_Settings_ tab
- Make sure the service is enabled
- *Interfaces* : LAN
- *Valid lifetime* : 4000, this corresponds to ~1hr 6min

![kea-settings](/screenshots/kea/kea-settings.png)


_Subnets_ tab
Click on the *+* button on the right end to add a new subnet and fill in as following:
- *Subnet* : OPNSense IP, i.e. 192.168.1.0/24
- *Description* : Dynamic IP Pool
- *Pools* : specify the dynamic IP pool, i.e. _192.168.1.200 - 192.168.1.254_
- Keep everything else as default

Once done, click on _Save_

![kea-subnets-pool](/screenshots/kea/kea-subnets-pool.png)


_Reservation_ tab
Click on the *+* button on the right end to add a new static IP device in the reservation pool and fill in as following:
- *Subnet* : it will be defaulted to the predefined subnet earlier (i.e. 192.168.1.0/24) unless you have more than one subnet, then select the needed
- *IP address* : specify the static IP of the device, i.e. 192.168.1.76
- *MAC address* : specify the device MAC address
- *Hostname* : name of the device, to make it easier to recognise with many defined devices. Make sure not to use capital letters, spaces, or underscores.

Once done, click on _Save_

Repeat the same steps above for each device. MAC address can be found in *Services-->Kea DHCP [new]-->Leases DHCPv4*


![kea-reservation](/screenshots/kea/kea-reservation.png)



# Dyn-DNS

This is a Dynamic DNS auto-updater, useful if you have your own domain that requires frequent public IP updates in case the ISP decides to change it. I use it with my Cloudflare tunnel

First, the plugin needs to be installed. Navigate to *System-->Firmware-->Plugins*. On that page look for the _os-ddclient_ plugin and click on the *+* sign at the end of it

![dyndns-plugin](/screenshots/dyndns/dyndns-plugin.png)


Once the installation is completed, navigate to *Services-->Dynamic DNS-->Settings*. On that page, at the top, click on the *General settings* tab and fill in the following:

- Make sure the account is enabled
- *Interval* : 300 (_can be set to 30 for testing, then make sure it is set to 300 when everything works_)
- *Backend* : ddclient

Once done, click on *Apply*

![dyndns-general-settings](/screenshots/dyndns/dyndns-general-settings.png)


On the same page, at the top, click on the *Accounts* tab, then click on the *+* button on the right end to add a new service, and fill in the following:

![dyndns-settings](/screenshots/dyndns/dyndns-settings.png)


On that page, fill in the following:
- Make sure the account is enabled
- *Service* : Cloudflare
- *Username* : email address of the account
- *Password* : Global API key for the account (*[Cloudflare](https://dash.cloudflare.com/)-->Account Overview-->Get your API token-->Global API Key--> View*)
- *Wildcard* : false
- *Zone* : your domain name, i.e. _mydomain.com_
- *Hostname* : same as the _Zone_ set above
- *Check IP method* : Interface
- *Interface to monitor*: WAN
- *Check ip timeout* : 10
- *Force SSL*: true

When done click on *Save*

![dyndns-add-new](/screenshots/dyndns/dyndns-add-new.png)


Now you will see that the service has been added, and if nothing is wrong, you will see the Current IP, and when it was updated. Make sure the _Enabled_ is ticked on the left side.

![dyndns-added](/screenshots/dyndns/dyndns-added.png)


If all is good, then click on *Apply* 



# Adguard Home 
#### [ref](https://www.max-it.de/en/adguard-dns-blocker-neues-opnsense-plugin/)


Adguard Home blocks ads, tracking, and cookies files. Personally, I prefer this as it gives the ability to easily define different and independant rules for each device connected to the network.

This is a plugin that needs to be installed, but first, the repo must be added by SSH into OPNSense with _root_ privileges (check #1 above for that), and once there, do the following:
#### [ref](https://github.com/mimugmail/opn-repo)

- Enter option *8* to open Shell
- execute the following: `fetch -o /usr/local/etc/pkg/repos/mimugmail.conf https://www.routerperformance.net/mimugmail.conf
pkg update`
#### To remove this repo later, first all installed packages from this repo must be removed by executing `pkg query -a '%R %n-%v' | grep mimugmail`, remove them from the UI, and then execute `rm /usr/local/etc/pkg/repos/mimugmail.conf`

Once successful, open OPNSense UI page and navigate to *System-->Firmware-->Plugins*. On that page look for the _os-adguardhome-maxit_ plugin and click on the *+* sign at the end of it.

![adguard-plugin](/screenshots/adguard/adguard-plugin.png)



Once the installation is completed, you need to enable the plugin first, by navigating to *Services-->Adguardhome-->General*, and from there make sure that _Enable_ is checked while _Primary DNS_ is un-checked

![adguard-enabled](/screenshots/adguard/adguard-enabled.png)


Now, open _Adguard_ home page by navigating to _192.168.1.1:3000_. Note this is the same IP as OPNSense but with an added port.

Follow the installation steps, which are 5 screens only. Note that there are 2 changes in the _second screen_ as follows:

_Screen 2_
Under the *_Admin Web Interface_*
- *Listen interface* : All interfaces
- *Port* : 3000

Under the *_DNS server_*
- *Listen interface* : All interfaces
- *Port* : 5310

![adguard-setup-2](/screenshots/adguard/adguard-setup-2.png)


After a successful setup, you will be prompted to log in to Adguard, do that and follow those steps to finalise the setup
#### A top message stating there is a new version and needs updating, do that as a first step

Navigate to *Settings-->General settings*
*Enable to following*
_General setting_
- Block domains using filters and hosts files
- Use AdGuard parental control
_Logs configuration_
- Enable log

Navigate to *Settings-->DNS settings*
_Upstream DNS servers_
- Select *Parallel requests*

Navigate to *Filters-->DNS blocklists*, and click on _Add blocklist_. Select what is needed (personally I selected them all)

#### In all lists were selected, and Facebook is not showing any comments on the feeds, then navigate to *Filters-->Custom filtering rules*, and add the below `@@||graph.facebook.com^$important`

Now, you can leave Adguard on the side and open OPNSense to _192.168.1.1_. From there, navigate to *Firewall-->NAT-->Port Forward*. On that screen, click on the *+* button on the right end to add a rule, and fill in the following:

- *Interface* : LAN
- *Protocol* : UDP
- *Source* : click on _Advanced_, then select _LAN net_
- *Destination* : any
- *Destination port range*: _from_ DNS _to_ DNS
- *Redirect target IP*: select _Single host or Network_ and put the OPNSense IP address, i.e. _192.168.1.1_
- *Redirect target port*: select _(other)_ and put in _5310_
- Keep everything else as default

When done click on *Save* and then click on *Apply* 

![adguard-opnsense-port-forward-1](/screenshots/adguard/adguard-opnsense-port-forward-1.png)
![adguard-opnsense-port-forward-2](/screenshots/adguard/adguard-opnsense-port-forward-2.png)



## GeoIP Blocking

The GeoIP blocking is used to block outside network entrances using GeoIP lists. There few providers for those lists, but the most popular is the MaxMind GeoLite 2 which is a free service.

Head to their website and follow the steps to [signup](https://www.maxmind.com/en/geolite2/signup). After this, on the left side panel, navigate to *Account-->Manage License Keys--> Generate new licence key*


![geoip-licence-key](/screenshots/geoip/geoip-licence-key.png)

Name this key whatever you want, i.e. _OPNSense_, and in the next screen, take note of the generated key as this is the only time you will be able to see it, and keep it in a safe place for later usage.


Head now to the OPNSense portal and navigate to *Firewall-->Settings-->Advanced*. Scroll down (almost to the end) till you find the _Firewall Maximum Table Entries_ and insert the value of _2000000_ (it is an empty value that defaults to 1000000, and this is not enough for the GeoIP to handle)

![geoip-max-entries-table](/screenshots/geoip/geoip-max-entries-table.png)


Now, navigate to *Firewall-->Aliases*, and go to the *GeoIP settings*. In the _Url_ insert the following link
`https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country-CSV&license_key=<YOUR_GENERATED_LICENCE_KEY>&suffix=zip` and replace _<YOUR_GENERATED_LICENCE_KEY>_ by the key obtained in the first step.

When done click on *Apply*. On the upper right corner, there should be a green bar with around 48% usage, this means the lists were downloaded and applied on the OPNSense

![geoip-url](/screenshots/geoip/geoip-url.png)


Now, on the same screen, click on the *Aliases* tab and click on the *+* sign at the end of it, and fill in the following:

- Make sure the alias is enabled
- *Name* : whatever you want as a name, i.e. _block-continents_
- *Type* : _GeoIP_ and next to it select _IPv4, IPv6_
- *Content* : select the countries from each Region as needed, or simply click on the _tick_ on the side to select all countries in that region
- Enable *Statistics*
- *Description* : Blocks all Continents

When done click on *Apply*

![geoip-new-alias](/screenshots/geoip/geoip-new-alias.png)


Last thing, we need to define a firewall rule to use those blocking lists. Navigate to *Firewall-->Rules-->WAN* and click on the *+* sign at the end of it, and fill in the following:

- *Action* : Block
- *Interface* : WAN
- *Direction* : in
- *TCP/IP version* : IPv4+IPv6
- *Protocol* : any
- *Source* : block-continents (the list created earlier in the alias above)
- *Destination* : any
- *Description* : GeoIP Blocking Rule
- Keep everything else as default

When done click on *Save* and then click on *Apply* 

![geoip-firewall-rule](/screenshots/geoip/geoip-firewall-rule.png)


*Make sure that this new firewall rule is at the top of the list (not considering the Automatically generated rules of course)*



## ClamAV

This is an antivirus engine for detecting malicious threats. 

To enable this, the plugin needs to be installed. Navigate to *System-->Firmware-->Plugins*. On that page look for the _os-clamav_ plugin and click on the *+* sign at the end of it

![clamav-plugin](/screenshots/clamav/clamav-plugin.png)


Next, it needs to be enabled by accessing OPNSense as _ROOT_ using the terminal as in step 1 above, then execute the following commands one at a time (those will take some time to finalise, so do not worry)

- *Download initial signatures*
`freshclam`

- *Enable and start clamAV-daemon*
`/usr/local/etc/rc.d/clamav_clamd enabled`
`/usr/local/etc/rc.d/clamav_clamd start`

- *Enable and start freshclam-daemon*
`/usr/local/etc/rc.d/clamav_freshclam enabled`
`/usr/local/etc/rc.d/clamav_freshclam start`

![clamav-cli](/screenshots/clamav/clamav-cli.png)


The last step is to make sure everything is working by navigating to *Services-->ClamAV-->Configuration*, just make sure it is enabled and leave everything as default

![clamav](/screenshots/clamav/clamav.png)



## Spamhaus Lists

This enables OPNsense firewall's mail security solutions for spam filtering. To do so, navigate to *Firewall-->Aliases* and click on the *+* sign at the end of it, and fill in the following:

- Make sure the alias is enabled
- *Name* : spamhaus_drop
- *Type* : URL Table (IPs)
- *Refresh Frequency* : 1 Days 0.00 Hours
- *Content* : https://www.spamhaus.org/drop/drop.txt
- *Description* : Spamhaus DROP
- Keep everything else as default

When done click on *Save* and then click on *Apply* 

![spamhaus_drop](/screenshots/spam_lists/spamhaus_drop.png)


Repeat the same by adding another alias for the EDROP list and fill in the following:
- Make sure the alias is enabled
- *Name* : spamhaus_edrop
- *Type* : URL Table (IPs)
- *Refresh Frequency* : 1 Days 0.00 Hours
- *Content* : https://www.spamhaus.org/drop/edrop.txt
- *Description* : Spamhaus EDROP
- Keep everything else as default

When done click on *Save* and then click on *Apply* 

![spamhaus_edrop](/screenshots/spam_lists/spamhaus_edrop.png)


Now, navigate to *Firewall-->Rules-->WAN* and click on the *+* sign at the end of it, and fill in the following:
- *Action* : Block
- *Interface* : WAN
- *Direction* : in
- *TCP/IP version* : IPv4+IPv6
- *Protocol* : any
- *Source* : spamhaus_drop
- *Destination* : any
- *Category* : Spamhaus rules
- *Description* : Block Spamhaus DROP
- Keep everything else as default

When done click on *Save* and then click on *Apply* 


Do the same for the EDROP list
- *Action* : Block
- *Interface* : WAN
- *Direction* : in
- *TCP/IP version* : IPv4+IPv6
- *Protocol* : any
- *Source* : spamhaus_edrop
- *Destination* : any
- *Category* : Spamhaus rules
- *Description* : Block Spamhaus EDROP
- Keep everything else as default

When done click on *Save* and then click on *Apply* 

![firewall_wan](/screenshots/spam_lists/firewall_wan.png)


Now, navigate to *Firewall-->Rules-->LAN* and click on the *+* sign at the end of it, and fill in the following:
- *Action* : Block
- *Interface* : LAN
- *Direction* : in
- *TCP/IP version* : IPv4
- *Protocol* : any
- *Source* : spamhaus_drop
- *Destination* : any
- *Category* : Spamhaus rules
- *Description* : Block Spamhaus DROP
- Keep everything else as default

When done click on *Save* and then click on *Apply* 

Do the same for the EDROP list
- *Action* : Block
- *Interface* : LAN
- *Direction* : in
- *TCP/IP version* : IPv4
- *Protocol* : any
- *Source* : spamhaus_edrop
- *Destination* : any
- *Category* : Spamhaus rules
- *Description* : Block Spamhaus EDROP
- Keep everything else as default

When done click on *Save* and then click on *Apply* 

![firewall_lan](/screenshots/spam_lists/firewall_lan.png)



## License
This document guide is licensed under the CC0 1.0 Universal license. The terms of the license are detailed in [LICENSE](/LICENSE)
