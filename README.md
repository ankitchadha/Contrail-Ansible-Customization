# Contrail-Ansible-Customization

## NTP use case:

#### Step-1: Update the schema:

Add this dictionary to fabric_config_schema.json. Note that this must be added at the same hierarchy as SNMP or static-routes (these are all dictionaries as well) :

              "ntp": {
                "type": "object",
                "additionalProperties": false,
                "properties": {
                  "time_zone": {
                  "type": "string"
                 }
               }
              },

Dynamic GUI rendering for such additional features is not supported in 5.0.1. As a result, the user cannot use the Contrail-Command UI to provide the input for the NTP server. An alternative is to use the VNC APIs to get this done.

#### Step-2: Update the role-config object using the VNC APIs:

I used a setup with vQFXs, these correspond to the qfx10k role-config object.

from vnc_api import vnc_api\
vnc_lib = vnc_api.VncApi()\
vnc_lib = vnc_api.VncApi()\
rc_obj = vnc_lib.role_config_read(id="73f6bf3b-68e9-484c-93c1-81c5bb6cdf22") << UUID For QFX10k role-config\
rc_obj.dump()\
------------ role-config ------------\
Name =  [u'default-global-system-config', u'juniper-qfx10k', u'underlay_ip_clos']\
Uuid =  73f6bf3b-68e9-484c-93c1-81c5bb6cdf22\
Parent Type =  node-profile\
*#P role_config_config =  {u'bgp_hold_time': 90}#* ## This is the value that needs to be changed\
P id_perms =  permissions = owner = cloud-admin, owner_access = 7, group = cloud-admin-group, group_access = 7, other_access = 7, uuid = uuid_mslong = 8356076420516628556, uuid_lslong = 10646933680333578018, enable = True, created = 2018-11-07T02:47:45.631829, last_modified = 2018-11-07T02:47:46.015576, description = None, user_visible = True, creator = None\
P perms2 =  owner = cloud-admin, owner_access = 7, global_access = 0, share = []\
P annotations =  None\
P display_name =  None\
REF job_template =  [{u'to': [u'default-global-system-config', u'fabric_config_template'], u'href': u'http://172.16.1.102:8082/job-template/7b0e099e-45a0-43b8-8f66-b50be2ae2fcc', u'attr': None, u'uuid': u'7b0e099e-45a0-43b8-8f66-b50be2ae2fcc'}]\
REF tag =  None\


IMPORTANT NOTE: The stock role_config_config value is a dictionary, but at the time of customization the user needs to provide the value as a string. And that string should be in a valid dictionary format. This dictionary format should conform to the schema changes that were made earlier. Check it out:

rc_obj.set_role_config_config("{'ntp': {'time_zone': 'PST'}, 'snmp': {'communities': [{'readonly': true, 'name': 'public'}]}}")
vnc_lib.role_config_update(rc_obj)

Now you should be able to see your updated changes if you query the Contrail-API server (port# 8082).

#### Step-3: Update the Jinja template

CEM's Ansible playbooks use jinja2 templates to generate the Junos configuration for the underlay fabric devices. Since NTP is a basic use-case, add the following lines to the juniper_basic.j2 file:

{%         if feature_params.ntp is defined and feature_params.ntp.time_zone is defined %}
set groups {{cfg_group}} system time-zone {{ feature_params.ntp.time_zone }}
{%         endif %}

#### Step-4: Discover the fabric

After discovering the fabric, you'll see the corresponding NTP configuration being pushed on the underlay QFX switch:

vagrant@vqfx1# show groups __contrail_basic__ | display set
set groups __contrail_basic__ system time-zone PST
set groups __contrail_basic__ snmp community public authorization read-only

{master:0}[edit]

NOTE: All the configuration pushed by contrail to the underlay devices is stored under a Junos configuration group. This means any more-specific user-configuration is not impacted by Contrail's discovery of the underlay Fabric devices.

## BFD use-case:
The idea is the same with a slight difference. For my demo/design, BFD is needed only for the Underlay BGP peering. The idea is that if the underlay peering goes down, it'll automatically take down the overlay peering.
Some differences from the use-case mentioned above:
1. Schema change will need to be done to the ip_clos section
2. jinja2 template for the ip_clos functionality will need to be changed

Detailed steps with snippets to follow soon.
