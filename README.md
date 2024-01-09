# Edgerouter IPSEC VTI FQDN (for Dynamic IP)
This script provides a workaround to allow you to maintain IPSEC VTI-base site-to-site VPNs based on FQDNs. It is especially useful where one or more of your ISPs provides you with a dynamic IP address rather than static.

It should be installed on any Edgerouter on your network which is at one end of such a VPN between Edgerouters and will automatically update the ER config at each end of the IP address for the FQDN if one of them changes.

It only requires non-invasive changes to the ER config and copying scripts onto the Edgerouter. There is deliberately no need to install separate packages etc.

I have it running on 3 ERs (X, 12 and 12P) in 3 locations with a VPN between each pair. Since one is in a different country to the other two, and the latter two are 2.5 hours drive apart, that's quite handy!
## How it Works
Each Edgerouter must, of course, have its IP address mapped to a FQDN - you can use DuckDNS, Cloudflare or any other Dynamic DNS service to maintain that for you.

It requires the description field of each peer in the Edgerouter's config to be set to the FQDN of that remote peer's Edgerouter.

It assumes that all VPNs connect via the WAN address of the router.

A local script is installed on the Edgerouter which can be invoked in one of two ways:
- every few mins via a cron job
- triggered off an event from something logged in one of the log files (or syslog)

This script checks the WAN IP address of the local router and the IP address of the peer's FQDN to see if they differ from what the vpn ipsec config is set to and, if so, update the config with the corrections and commits and saves it.

Both ends of a VPN will get triggered to do this, and the tunnel will come backup.

There are extensions for externally-proxied IP addresses (see below)
## Installation
Take all the scripts in this repo (you only actually need the first one unless you are dealing with a proxying provider like Cloudflare):
- `maintain_ipsec_vti_vpn_fqdn`
- `get_domain_ip_cloudflare
- `get domain_ip_cloudflare.conf
- `ini-file-parser.sh`
and copy them to a directory on the local Edgerouter (I put it in `/config/scripts` as per the guidance as it is untouched by upgrades). In that directory, run (if required):
```
chmod a+x maintain_ipsec_vti_vpn_fqdn get_domain_ip_cloudflare
```
If you do install it somewhere else (all files need to be in the same directory), edit the following line in `maintain_ipsec_vti_vpn_fqdn` as appropriate, but why bother?
```
export INSTALL_DIR=/config/scripts
```
In the Edgerouter's main config, modify (or add) the description field for each ipsec site-to-site peer you have, as follows, replacing the ip address and FQDN as appropriate:
```
configure
set vpn ipsec site-to-site peer xx.xx.xx.xx description example.domain.com
(repeat for each such peer on this Edgerouter)
commit; save
exit
```
Firstly, I suggest running the script `maintain_ipsec_vti_vpn-fqdn` manually to check it works ok. This script take no arguments but has three optional flags you can set:
- `-v` to print out the config change commands that are going to be applied and the ones it is replacing
- `-i` if no changes are required in a run, still log confirmation of the run
- `-d` to do a dry run: no changes will be applied
In this initial run, suggest running:
```
./maintain_ipsec_vti_vpn_fqdn -vid
```
You can run the script manually to make sure it doesn't throw any errors - it should finish and confirm nothing needs to be changed.

You can also do more comprehensive testing as you see fit as detailed below.

Note that stdout and stderr are not redirected to the log file if you run it interactively

You then need to setup how it is triggered on the Edgerouter choosing one of the Trigger sections below.

Once you have got it done on your local Edgerouter, you can do the same on each remote - any that are at one end of an IPSEC VTI VPN.

You can do the testing on those remotes also (just don't test with the vpn you are connecting by!)
### Trigger on a schedule
Run the Edgerouter command to effectively add something to crontab with:
```
configure
set system task-scheduler task checkVPNs interval 5m
set system task-scheduler task checkVPNs executable path /config/scripts/maintain_ipsec_vti_vpn_fqdn
set system task-scheduler task checkVPNs executable arguments -vid
commit; save
exit
```
You can check what this has set in teh crontab by doing `cat /etc/cron.d/vyatta-crontab`

Note that it has `-vid` - logging everything as a dryrun first. You can check the logs at `var/log/maintain_ipsec_vti_fqdn.log` and, when you are happy, take off the `d` by setting that config line again without it. You may also remove the `i` if you are concerned about the log filling up with one line per run per VPN. The `v` is probably OK - up to you: it will only log something if there is a change and that should not be too frequent so probably no harm.
### Trigger from VPN event
Still need to finish working this out
## Exhaustive Testing in Your Environment
It's not great if this goes wrong! My suggestion is to test it and the following does a comprehensive test of all change scenarios (this will momentarily interrupt connectivity for one VPN make sure that's OK)

Make sure the description field config has been applied as above and any Cloudflare or other proxy-provider has been made - these are non-invasive and can be safely changed.

When you have run these tests or however many you want to do, you should be good to go to copy all files including any changed proxy-provided config to each remote Edgerouter also as part of setting up each of those.

***Embarrassing word of warning*** - if you feel the need to test, don't do what I did in my early testing (where of course things could go wring as I iterated on the code) and do your first installation and test on a remote Edgerouter. That is the IT equivalent of chainsawing off of a tree the branch you are sitting in. Yep - I would be laughing at someone else if they did this too.
### Local-Address Changed
- Change your config for one of  the peers in your config to have the local address set to something different - a `fake-address`
- Commit this - it will complain that no interface is configured with that address but, it still saves the config anyway.
- Run the script as `./maintain_ipsec_vti_vpn_fqdn -vid` and you should see it report that this peer needs a change, showing you an existing config and new config as:
	```
	(existing)
	set vpn ipsec site-to-site xx.xx.xx.xx local-address <fake-address>
	(new)
	set vpn ipsec site-to-site xx.xx.xx.xx local-address <original-address>
	```
- then run it properly as `./maintain_ipsec_vti_vpn_fqdn -vi`
- Check your config to see that it has reverted.
### Remote-Address Changed
- Recreate the for config for one of the peers in your config to change the peer to something different (NOT an ip address that already has another peer defined for it!). Bets way to do this is to run at the shell level outside of the configure facility:
	```
	show configuration commands | grep "^set vpn ipsec site-to-site peer <original-address>"
	```
- Copy this command in an editor, change `<original-address>` to `<fake_address>` in all lines
- Add a line to the front of this config:
	```
	delete vpn ipsec site-to-site peer <original-addresss>
	```
- Paste into a configure session, make sure there are no errors and `commit` 
- Run the script as `./maintain_ipsec_vti_vpn_fqdn -vid` and you should see it report that this peer needs a change, showing you an existing config and new config as:
	```
	(existing)
	set vpn ipsec site-to-site <fake-address> ...
	set vpn ipsec site-to-site <fake-address> ...
	etc
	(new)
	set vpn ipsec site-to-site <original-address> ...
	set vpn ipsec site-to-site <original-address> ...
	etc
	```
	the new and old config should be identical except for the peer  n run it properly as `./maintain_ipsec_vti_vpn_fqdn -vi`
- Check your config to see that it has reverted.
### Both Local-Address and Remote-Address Changed at the Same Time
- Follow the process for the remote address change but, when you paste and create the script also change the local-address line as per the section before
- When you run it in dryrun mode the first time, you should see:
	```
	(existing)
	set vpn ipsec site-to-site <remote-fake-address> ...
	set vpn ipsec site-to-site <remote-fake-address> ...
	...
	set vpn ipsec site-to-site <remote-fake-address> local-address <local-fake-address>
	(new)
	set vpn ipsec site-to-site <remote-original-address> ...
	set vpn ipsec site-to-site <remote-original-address> ...
	...
	set vpn ipsec site-to-site <remote-original-address> local-address <local-original-address>
	```
	That really *should* be one less line, but it was easier in the code, for now, to do the remote check and the local check serially and you'll get two local-address lines, but that's ok, as lon as they are done in teh riht order, it will be fine.
## Usage
That's it, it should just automatically fix dynamic IP issues with IPSEC VTI tunnels or static ones that you change and want this to pick up.

A log of each time a change was detected and the result can be found at `/var/log/maintain_ipsec_vti_vpn_fqdn.log

One thing it does not cover is where you swap the IP addresses of two VPN-connected routers at the same time. Why you would do this I don't know, but if you do, the script will (should) fail with config errors since you would be trying to create an identical peer to one that already exists. But, it's an edge case that will never happen unless you choose to manually do something as silly as this.
## Externally-Proxied Domains (eg Cloudflare)
Some DNS services that you register your domain which proxy your Edgerouters WAN IP address so, when you do a DNS lookup on that domain, it returns instead the IP address of an external proxy.

I've written a script to handle this for Cloudflare which is incorporated into this as follows:

When you edit the description field in the config for such a VPN, append `:cloudflare` to the FQDN, so that the config change line looks similar too:
```
set vpn ipsec site-to-site peer xx.xx.xx.xx description example.domain.com:cloudflare
```
Also, you will need to add your Cloudflare access info to `get domain_ip_cloudflare.conf` for each domain (copy/paste if they are the same). You can including all FQDNs that participate in a VPN across your whole network and then you can just copy the same config to all Edgerouters:
```
[example.com]          ## must match the FQDN you have entered into the description field above
API_KEY = yourapikey   ## the Global API key from the Cloudflare profile owning the domain
EMAIL   = youremail    ## thethe login email of the Cloudflare profile owning the domain
ZONE_ID = zoneid       ## the unique identifier for the domain in question from Cloudflare Overview page
```
When it encounters a `:` after the FQDN, it takes what is after that (in this case `cloudflare` and calls a script called get_domain_ip_`cloudflare` which then uses `ini_file_parser.sh` to read the get_domain_ip_`cloudflare`.conf file to access the Cloudflare API and return the IP address you are looking for.
## Adding your own External Proxy Provider 
You can add scripts and config for other proxied domain providers: follow the same pattern:
- Select a label for this provider similar to `cloudflare` - let's call it `xxxxx`
- Create a config file `get_domain_ip_xxxxx.conf` 
- Create a script called `get_domain_ip_xxxxx` which returns back a single IP address, taking a FQDN as argument and using this dedicated config file.

If you create this for a new provider, create a pull request here and let's add the script and dummy config file so that other can use with that provider (and update README of course)
## To Do
- Finalise the event trigger method
- Change so that it will work also where VPNs go over a non-WAN interface(?)
- Accommodate other proxy providers (but I will leave that to others to contribute)
## Contributors
First time I used ChatGPT to get most of the way in writing a script initially. Lots of iterations and discussion with in. I had to finish it off and then do a significant amount of extension but it was pretty impressive (and scary) - I found myself dipping into it then for bits and pieces as I evolved it further.

I also copied over `ini_file_parser.sh` as-is from https://github.com/DevelopersToolbox/ini-file-parser so cheers for that.

On the journey, I also discovered how to run vyatta config commands from a bash script. The pain and suffering of that is described in this post: [TBD] 

Feel free to contribute to this via the usual pull request process
## License
[WTFPL](https://choosealicense.com/licenses/wtfpl/) except for `ini-file-parser.sh` (details for that script at its source: https://github.com/DevelopersToolbox/ini-file-parser)
