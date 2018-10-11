## FAQs
* Basics of [Concourse](https://concourse-ci.org/)
* Basics of running [Concourse using docker-compose](https://github.com/concourse/concourse-docker)
* Basics of the pipeline functioning
  * Check the blog post: [ Introducing nsx-t-gen: Automating NSX-T Install with Concourse](https://allthingsmdw.blogspot.com/2018/05/introducing-nsx-t-gen-automating-nsx-t.html)
* `Adding additional edges after first install`.
  * Recommend planning ahead of time and creating the edges all in the beginning rather than adding them later.
  * If its really required, recommend manually installing any additional edges using direct deployment of OVAs while ensuring the names are following previously installed edge instance name convention (like nsx-t-edge-0?), then update the parameters to specify the additional edge ips (assuming they use the same edge naming convention) and let the controller (as part of the base-install or just full-install) to do a rejoin of the edges followed by other jobs/tasks. Only recommended for advanced users who are ready to drill down/debug.
* Downloading the bits
  * Download NSX-T 2.22 bits from
    https://my.vmware.com/group/vmware/details?downloadGroup=NSX-T-220&productId=673
    Check https://my.vmware.com for link to new installs
  * Download [VMware-ovftool-4.2.0-5965791-lin.x86_64.bundle v4.2](https://my.vmware.com/group/vmware/details?productId=614&downloadGroup=OVFTOOL420#)

  Ensure ovftool version is 4.2. Older 4.0 has issues with deploying ova images.
* Installing Webserver
  Install nginx and copy the bits to be served
	```
	# Sample nginx server to host bits
	sudo apt-get nginx
	cp <*ova> <VMware-ovftool*.bundle> /var/www/html
	# Edit nginx config and start
	```
* Unable to reach the webserver hosting the ova bits
  * Check for a web proxy interfering with the concourse containers.
  If using docker-compose, use the sample [docker-compose](./docker-compose.yml) template to add DNS and proxy settings. Add the webserver to the no_proxy list.

  Ensure you are using docker-compose version `1.18+` and docker compose file version is `3`

  Check with docker documentation on specifying proxies: https://docs.docker.com/network/proxy/

  Ensure the `/etc/systemd/system/docker.service.d/http-proxy.conf` specifies the HTTP_PROXY and HTTPS_PROXY env variables so docker can go out via the proxy.
  ```
  [Service]
  Environment="HTTP_PROXY=http://proxy.corp.local"   # EDIT the proxy
  Environment="HTTPS_PROXY=http://proxy.corp.local"  # EDIT the proxy
  Environment="NO_PROXY=localhost,127.0.0.1,<local-vm-ip>"
  ```

  Stop the docker service, reload the daemons and then start back the docker service
  ```
  systemctl stop docker
  systemctl daemon-reload # to reload the docker service config
  systemctl start docker
  ```

  Or use the ~/.docker/config.json approach to specify the proxy.

  * Disable ubuntu firewall (ufw) or relax iptables rules if there was usage of both docker concourse and docker-compose.
    Change ufw
  	```
	sudo ufw allow 8080
	sudo ufw default allow routed
	```
	or relax iptables rules
	```
	sudo iptables -P INPUT ACCEPT
	sudo iptables -P FORWARD ACCEPT
	sudo iptables -P OUTPUT ACCEPT
	```

* If running out of disk space with docker compose, use `docker volume prune` command to clean up unused volumes.

* If things are still not reachable to outside (like reaching the github repos or webserver), try to add an additional docker image to run alongside concourse like an vanilla ubuntu image for debug purpose and shell into it, then try to run a curl to outside after updating apt-get and installing curl.

Sample entry for adding ubuntu docker container image to docker-compose.yml.
```
services:
  # new ubuntu docker image
  ubuntu:
    image: ubuntu:17.10
    command: sleep 600000

  concourse-web:
    .....  
```
Find the docker container for the ubuntu image using `docker ps`
Then shell into it using `docker exec -it <container-id> /bin/bash`
Run following and see fi it can connect to outside via the proxy:
```
apt-get update -y && apt-get install -y curl
curl www.google.com
```
If the above curl command works but concourse is still not able to go out, then check the various `CONCOURSE_*` env variables specified for the proxy and garden and dns settings.

* Pipeline exits after reporting problem with ovas or ovftool
  * Verify the file names and paths are correct. If the download of the ovas by the pipeline at start was too fast, then it means errors with the files downloaded as each of the ova is upwards of 500 MB.
* Running out of memory resources on vcenter
  * Turn off reservation
  ```
  nsx_t_keep_reservation: false # for POC or memory constrained setup
  ```
* Install pipeline reports the VMs are unreachable after deployment of the OVAs and creation of the VMs.
  Sample output:
  ```
	Deployment of NSX Edge ova succcessfull!! Continuing with rest of configuration!!
	Rechecking the status and count of Mgr, Ctrl, Edge instances !!
	All VMs of type NSX Mgr up, total: 1
	All VMs of type NSX Controller up, total: 3
	All VMs of type NSX Edge down, total: 2
	 Would deploy NSX Edge ovas

	Some problem with the VMs, one or more of the vms (mgr, controller, edge) failed to come up or not accessible!
	Check the related vms!!
  ```
  If the vms are correctly up but suspect its a timing issue, just rerun the pipeline task.
  This should detect the vms are up and no need for redeploying the ovas again and continue to where it left of earlier.

  If the vms appear to be not reachable over ssh and they are on same host, problem might be due to known issue: https://kb.vmware.com/s/article/2093588
  ```Deploying a high number of virtual machines at the same time results in the network adapter connection failure and reports the error: Failed to connect virtual device Ethernet0 (2093588)```
  Reboot the esxi host thats hosting the vms and rerun the pipeline.
* Unable to deploy the Edge OVAs with error message: `Host did not have any virtual network defined`.
  * Refer to [add-vm-network](./add-vm-network.md)
  * Or deploy the ovas directly ensuring the name of the edge instances follows the naming convention (like nsx-t-edge-01)
* Unable to add ESXi Hosts. Error: `FAILED - RETRYING: Check Fabric Node Status` with error during ssh connection to the hosts.
  * Empty the value for `esxi_hosts_config` and fill in `compute_vcenter_...` section in the parameter file.
  	```
	esxi_hosts_config: # Leave it blank

    # Fill following fields
	compute_vcenter_manager: # FILL ME - any name for the compute vcenter manager
	compute_vcenter_host:    # FILL ME - Addr of the vcenter host
	compute_vcenter_usr:     # FILL ME - Use Compute vCenter Esxi hosts as transport node
	compute_vcenter_pwd:     # FILL ME - Use Compute vCenter Esxi hosts as transport node
	compute_vcenter_cluster: # FILL ME - Use Compute vCenter Esxi hosts as transport node
  	```
   Apply the new params using set-pipeline and then rerun the pipeline.
* Error during adding ESXi Hosts as Fabric nodes.
  Error message: ```mpa_connectivity_status_details : Client has not responded to 2 consecutive heartbeats,```
  Check the NSX Manager Web UI and see if the Hosts got added as Fabric Nodes (under Fabric -> Hosts) after some delay.
  If the hosts now appear healthy and part of the Fabric on the NSX Mgr, then retry the add-routers job in concourse and it should proceed to the remaining steps.
* Use different Compute Manager or ESXi hosts for Transport nodes compared vCenter used for NSX-T components
  * The main vcenter configs would be used for deploying the NSX Mgr, Controller and Edges.
    The ESXi Hosts for transport nodes can be on a different vcenter or compute manager. Use the compute_vcenter_... fields or esxi_hosts_config to add them as needed. Caution: If the NSX Edges are really on a completely different network compared to the Hosts, then its suboptimal as the Edge has to be the gateway for the overlay/tep network with the hosts.
* Control/specify which Edges are used to host a given T0 Router.
  * Edit the edge_indexes section within T0Router definition to specify different edge instances.
    Index starts with 1 (would map to nsx-t-edge-01).
  ```
  nsx_t_t0router_spec: |
  t0_router:
    name: DefaultT0Router
    ha_mode: 'ACTIVE_STANDBY'
    # Specify the edges to be used for hosting the T0Router instance
    edge_indexes:
      # Index starts from 1 -> denoting nsx-t-edge-01
      primary: 1   # Index for primary edge to be used
      secondary: 2 # Index for secondary edge to be used
    vip: 10.13.12.103/27
    ....
  ```
* Adding additional T1 Routers or Logical Switches
  * Modify the parameters to specify additional T1 routers or switches and rerun add-routers.
* Adding additional T0 Routers
  * Only one T0 Router can be created during a run of the pipeline. But additional T0Routers can be added by  modifying the parameters and rerunning the add-routers and config-nsx-t-extras jobs.
    * Create a new copy or edit the parameters to modify the T0Router definition (it should provide index reference to nsx-t edges thats not used actively or as backup by another T0 Router).
    * Edit T0Router references across T1 Routers as well as any tags that should be used to identify a specific T0Router.
    * Add or edit any additional ip blocks or pools, nats, lbrs
    * Register parameters with the pipeline
    * Rerun add-routers followed by config-nsx-t-extras job group

* Static Routing for NSX-T T0 Router
  Please refer to the [Static Routing Setup](./static-routing-setup.md) for details on the static routing.

* Errors with NAT rule application
  Sample error1: `[Routing] Service IPs are overlapping with logical router ports`
  Sample error2: `[Routing] NAT service IP(s) overlap with HA VIP subnet`
  If the external assigned ip used as a SNAT translated ip falls in the Router uplink port range (like T0 router is using /27 and the specified translated ip falls within the /27 range), then the above errors might get thrown. Restrict or limit the cidr range using something like /29 (configured in the T0 spec) that limits it to just 6 ips and use an external ip thats outside of this uplink router ip range as translated ip.

  Sample:
  ```
  nsx_t_t0router_spec: |
  t0_router:
    name: DefaultT0Router
    ...
    vip: 10.13.12.103/29  # T0 router vip - make sure this range does not intrude with the external vip ranges
    ip1: 10.13.12.101/29  # T0 router uplink ports - make sure this range does not intrude with the external vip ranges
    ip2: 10.13.12.102/29  # T0 router uplink ports - make sure this range does not intrude with the external vip ranges
   ```
  And external ip:
  ```
  nsx_t_external_ip_pool_spec: |
  external_ip_pools:
  - name: snat-vip-pool-for-pas
    cidr: 10.100.0.0/24  # Should be a 0/24 or some valid cidr, matching the external exposed uplink
    gateway: 10.100.0.1
    start: 10.100.0.31 # Should not include gateway, not overlap with the T0 router uplink ips; reserve some for Ops Mgr, LB Vips for GoRouter, SSH Proxy
    end: 10.100.0.200  # Should not include gateway, not overlap with the T0 router uplink ips
    # Specify tags with PAS 2.0 and NSX Tile 2.1.0
  ```
  And nat rule:
  ```
  nsx_t_nat_rules_spec: |
  nat_rules:
  # Sample entry for PAS Infra network SNAT - egress
  - t0_router: DefaultT0Router
    nat_type: snat
    source_network: 192.168.1.0/24      # PAS Infra network cidr
    translated_network: 10.100.0.12      # SNAT External Address for PAS networks, outside of the T0 uplink ip range
    rule_priority: 8000   
  ```
