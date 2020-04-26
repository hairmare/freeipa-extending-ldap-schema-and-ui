# extending-freeipa
An example to extend freeipa with custom attributes which can be configured through cli or web ui.

## Introduction
We needed this to integrate Owncloud/Nextcloud into FreeIPA. We wanted to be able to manage which user have Owncloud/Nextcloud shares and to set a quota individually. The latest documentation we found was for FreeiPA 3.3, [Extending FreeIPA](https://www.freeipa.org/images/5/5b/FreeIPA33-extending-freeipa.pdf). A lot has changed since then. This example here should work on FreeIPA 4 and later.

## How it works
First you'll have to extend the LDAP schema and then add some plugins for cli and web ui. We'll demonstrate by using the example for Owncloud/Nextcloud.

### Extending the schema
We used the object classes and attributes already defined for Nextcloud ([nextcloud.schema](https://github.com/nextcloud/univention-app/blob/master/nextcloud.schema)). We slightly adjusted them to fit our needs.
```
# Adjustments by $( 2020 ) Radio Bern RaBe, Simon Nussbaum <smirta@gmx.net>
# - Removed MUST ( cn ), because with this it did not work. But is not
#   a necessary condition for our case.
# - stripped objectClass 'nextcloudGroup', because it's not needed here.

# Attribute Types
#-----------------
dn: cn=schema
changetype: modify
add: attributeTypes
attributeTypes: ( 1.3.6.1.4.1.49213.1.1.1 NAME 'nextcloudEnabled'
        DESC 'whether user or group should be available in Nextcloud'
        EQUALITY caseIgnoreMatch
        SUBSTR caseIgnoreSubstringsMatch
        SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 SINGLE-VALUE)
attributeTypes: ( 1.3.6.1.4.1.49213.1.1.2 NAME 'nextcloudQuota'
        DESC 'defines how much disk space is available for the user'
        EQUALITY caseIgnoreMatch
        SUBSTR caseIgnoreSubstringsMatch
        SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 SINGLE-VALUE)
-
add: objectclasses
objectClasses: ( 1.3.6.1.4.1.49213.1.2.1 NAME 'nextcloudUser'
        DESC 'A Nextcloud user'
        SUP top AUXILIARY
        MAY ( nextcloudEnabled $ nextcloudQuota )
        )

```
https://github.com/radiorabe/extending-freeipa/blob/master/src/nextcloud.ldif


___important note: the ldif file has to end with a blank line___



### cli plugin
There are slight differences to the guide [Extending the FreeIPA Server](https://www.freeipa.org/images/5/5b/FreeIPA33-extending-freeipa.pdf), but it still works close enough this way. The main differences are the path to the plugins that has changed. The import path for 'user' has to be adjusted and all plugins have to be copied to ```<path to python lib>/ipaserver/plugins``` (e.g. ```/usr/lib/python2.7/site-packages/ipaserver/plugins```) instead of ```<path to python lib>/ipalib/plugins``` (e.g. ```/usr/lib/python2.7/site-packages/ipalib/plugins```).

Plugin to enable user for having a nextcloud share
```py
from ipaserver.plugins import user
from ipalib.parameters import Bool
from ipalib import _

user.user.takes_params = user.user.takes_params + (
    Bool('nextcloudenabled?',
        cli_name='nextcloudenabled',
        label=_('Nextcloud Share enabled?'),
        doc=_('Whether or not a nextcloud share is created for this user (default is false).'),
        default=False,
        autofill=True,
        ),
    )

user.user.default_attributes.append('nextcloudenabled')

def useradd_precallback(self, ldap, dn, entry, attrs_list,
                        *keys, **options):
    entry['objectclass'].append('nextclouduser')
    return dn

user.user_add.register_pre_callback(useradd_precallback)

def usermod_precallback(self, ldap, dn, entry, attrs_list,
                        *keys, **options):
    if 'objectclass' not in entry.keys():
        old_entry = ldap.get_entry(dn, ['objectclass'])
        entry['objectclass'] = old_entry['objectclass']
    entry['objectclass'].append('nextclouduser')        
    return dn

user.user_mod.register_pre_callback(usermod_precallback)
```
https://github.com/radiorabe/extending-freeipa/blob/master/src/userncenabled.py


Plugin for setting a quota for a user
```py
from ipaserver.plugins import user
from ipalib.parameters import Str
from ipalib.text import _

user.user.takes_params = user.user.takes_params + (
    Str('nextcloudquota?',
        cli_name='nextcloudquota',
        label=_('Nextcloud Share Quota'),
        doc=_('Defines Nextcloud share quota in Bytes. Allowed values are "none", "default", e.g. "1024 MB" (default is "default").'),
        default=u'default',
        autofill=true,
        pattern='^(default|none|[0-9]+ [MGT]B)$',
        pattern_errmsg='may only be "none", "default" or a number of mega-, giga- or terabytes (e.g. 1024 MB)',
        ),
    )

user.user.default_attributes.append('nextcloudquota')

def useradd_precallback(self, ldap, dn, entry, attrs_list,
                        *keys, **options):
    entry['objectclass'].append('nextclouduser')
    return dn

user.user_add.register_pre_callback(useradd_precallback)

def usermod_precallback(self, ldap, dn, entry, attrs_list,
                        *keys, **options):
    if 'objectclass' not in entry.keys():
        old_entry = ldap.get_entry(dn, ['objectclass'])
        entry['objectclass'] = old_entry['objectclass']
    entry['objectclass'].append('nextclouduser')
    return dn

user.user_mod.register_pre_callback(usermod_precallback)
```
https://github.com/radiorabe/extending-freeipa/blob/master/src/userncquota.py


### web ui plugin
To have input fields, radio buttons, check boxes in the web ui we have to add plugins for this as well. The plugins are written in java script.


Plugin to enable user for having a nextcloud share
```js
define([
		'freeipa/phases',
		'freeipa/user'],
		function(phases, user_mod) {
			
// helper function
function get_item(array, attr, value) {
	for (var i=0,l=array.length; i<l; i++) {
		if (array[i][attr] === value) 
			return array[i];
		}
		return null;
}

var nc_enabled_plugin = {};

// adds nextcloud enabled field into user account facet
nc_enabled_plugin.add_nc_enabled_pre_op = function() {	
	var facet = get_item(user_mod.entity_spec.facets, '$type', 'details');
	var section = get_item(facet.sections, 'name', 'account');
	section.fields.push({
				$type: 'checkbox', 
				name: 'nextcloudenabled', 
				label: 'Nextcloud Share enabled',
				flags: ['w_if_no_aci']
	});
	return true;	
};

phases.on('customization', nc_enabled_plugin.add_nc_enabled_pre_op);

return nc_enabled_plugin;
});
```
https://github.com/radiorabe/extending-freeipa/blob/master/src/userncenabled.js


Plugin for setting a quota for a user
```js
define([
		'freeipa/phases',
		'freeipa/user'],
		function(phases, user_mod) {
			
// helper function
function get_item(array, attr, value) {
	for (var i=0,l=array.length; i<l; i++) {
		if (array[i][attr] === value) 
			return array[i];
		}
		return null;
}

var nc_quota_plugin = {};

// adds nextcloud quota field into user account facet
nc_quota_plugin.add_nc_quota_pre_op = function() {	
	var facet = get_item(user_mod.entity_spec.facets, '$type', 'details');
	var section = get_item(facet.sections, 'name', 'account');
	section.fields.push({
				name: 'nextcloudquota', 
				label: 'Nextcloud Share Quota',
				flags: ['w_if_no_aci']
	});
	return true;	
};

phases.on('customization', nc_quota_plugin.add_nc_quota_pre_op);

return nc_quota_plugin;
});
```
https://github.com/radiorabe/extending-freeipa/blob/master/src/userncquota.js

## Installation
Assuming having cloned this repo
```
git clone git@github.com:radiorabe/extending-freeipa.git
cd extending-freeipa/src
``` 
### LDAP schema extension
Extend the schema with the following command on the or a FreeIPA server:
```
ldapadd -H ldap://$HOSTNAME -D 'cn=Directory Manager' -W -f nextcloud.ldif
```


Check if schema was extended:
```
ldapsearch -H ldap://$HOSTNAME -D 'cn=Directory Manager' -W -x -s base -b 'cn=schema' objectclasses | grep -i nextcloud
ldapsearch -H ldap://$HOSTNAME -D 'cn=Directory Manager' -W -x -s base -b 'cn=schema' attributetypes | grep -i nextcloud
```


Add the new object class to the ipa user object class:
```
ipa config-mod --addattr=ipaUserObjectClasses=nextcloudUser
```

### cli plugins
Copy the plugin files to ```<path to python libs>/ipaserver/plugins``` and restart apache.
```
cp usernc* /usr/lib/python2.7/site-packages/ipaserver/plugins/
cd /usr/lib/python2.7/site-packages/ipaserver/plugins/
python -m compileall usernc* && python -O -m compileall usernc*
apachectl graceful 
```

### web ui plugins
Copy the plugin files to a subfolder with the same name as the file in ```<freeipa ui root>/js/plugins/``` and restart apache.
```
mkdir /usr/share/ipa/ui/js/plugins/{userncenabled,userncquota}
cp userncenabled.js /usr/share/ipa/ui/js/plugins/userncenabled/
cp userncquota.js /usr/share/ipa/ui/js/plugins/userncquota/
apachectl graceful 
```
