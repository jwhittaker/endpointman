# FreePBX 16 endpointman module Fork

## Intro

This is a fork of Bill Simon's `endpointman` module release 16.0.0.1 for compatibility updates to work with FreePBX 16 running on php7.

FreePBX 16 VM was still throwing php7 related errors. This project is to patch those errors as they come up while working only with the Cisco/Linksys SPA8000 8-port ATA device for the provisioning.

- The [voip-info.org forum thread](https://www.voip-info.org/forum/threads/oss-epm-for-freepbx-16-ipbx-2027.26880/page-2#post-169285) of the announced code in Jan 2023. 
- [Bill Simon's endpointman fork](https://github.com/billsimon/endpointman)
- The endpointman on the [FreePBX github repo](https://github.com/FreePBX-ContributedModules/endpointman) only supports up to FreePBX 14.0.
- The [backend provisioner](https://github.com/provisioner/Provisioner) code from Feb 2020.

### Summary of the Fixes

The first thing accomplished was getting to the pages without errors:

- `admin/config.php?display=epm_devices`
- `admin/config.php?display=extensionsettings`

These fixes were performed:

1. When copying the downloaded directory over from the [github source](https://github.com/billsimon/endpointman), recursively change ownership of everything to the `asterisk` user instead of the root user. This will solve failing `chmod()` calls within the php code (seen in `page.epm_devices.php`). The same will have to be done for using this very fork. Commands are shown further along in this readme.
2. Re-enable the module in the UI settings to update to `16.0.0.1` (it will prompt for this).
3. Another issue to consider later on is how to handle the JSON interface for code maintainability. `Services_JSON` is from 2005 and JSON is now built into php.
4. Commented out a line in `includes/functions.inc` that was specifically using the legacy json lib `//require_once('json.inc');`. This has not been fully tested.
5. Commented out a line in `includes/ajax.inc` that also included the leagacy json lib `//include 'json.inc';`. Also not fully tested.
6. Changed `includes/rain.tpl.class.inc` constructor from the legacy php4 syntax to `function __construct()`.

### JSON Handling

Another issue to consider is how to handle the JSON interface for code maintainability.

1. The `includes/json.inc` syntax is the first thing to throw errors up in the endpointman's **Extension Mapping** UI controls. This is from the legacy `Services_JSON` class. It looks like changing all of the curly braces to index arrays differently would solve the deprecation errors. But this file is stamped in the comments that it is from 2005. [Further history search in the FreePBX repo](https://github.com/FreePBX-ContributedModules/endpointman/commit/7801e1e60d2742218471386c9bf6c1e300c5547c#diff-0b67be94277c5fee3f97ce158b5145dedd0589707c1c18ca33836d3a8a0421ee) shows it was last updated within there in 2012 with possibly a bug. The line `if(!function_exists('json_decode')) {` seems like it is supposed to wrap the entire library, and be the legacy fall-back library if the php built-in `json_decode()` fails. It's a guess that perhaps over 10 years ago that was not noticed since php back then likely gave a warning rather than a hard fail. But the file seems like it needs more indentation for clarity. Or it's not clear if it's the case it checks for built-in functions, but the legacy syntax error is still thrown. And then there's its later clone with the `lib/json.class.php` file.
2. [There are patch files for this library](https://core.trac.wordpress.org/ticket/47751) but the FreePBX include of the library has been so long ago modified that the patch probably would not work in-place. This could be another point for transitioning the code to use the built in json functionalities.
3. There's something new going on with `lib/json.class.php`. It appears to be a copy of `includes/json.inc` [carried over in Feb 2016](https://github.com/FreePBX-ContributedModules/endpointman/commit/6876bae67c5d33756216f3d5b13797993666cbc1#diff-84c1c0e92e1d53cb7ec416e0c2722db520707e7c29e35877768b084cd58db39c). This is what Bill Simon [updated in Jan 2023](https://github.com/billsimon/endpointman/commit/9b3a807970a983a5be35170029839bb3142444cc#diff-84c1c0e92e1d53cb7ec416e0c2722db520707e7c29e35877768b084cd58db39c) to php7 compatible syntax.

## Setup and Procedures

### VM & PHP Info

Reports -> PHP Info Page

- php 7.4 is used on RHEL 7.9, *(release is from September 29, 2020)*
- kernel `3.10.0-1127`

`http://192.168.101.2/admin/config.php?display=phpinfo` output:

```
PHP Version 7.4.16
System	Linux freepbx.sangoma.local 3.10.0-1127.19.1.el7.x86_64 #1 SMP Tue Aug 25 17:23:54 UTC 2020 x86_64
Build Date	Mar 2 2021 10:35:17
Build System	Red Hat Enterprise Linux Server release 7.9 (Maipo)
Build Provider	Remi's RPM repository <https://rpms.remirepo.net/>
Server API	Apache 2.0 Handler
```

### Disable the Default Commercial Endpoint Manager

First disable the commercial endpoint manager.

If this is not done first, after installing the OSS version there will be an error message in the UI under **Settings -> OSS Endpoint Manager** about running two different endpoint managers.

Disable the commercial endpoint manager so that it does not conflict with the OSS one. There are several modules which also need to be disabled. The order of these commands matter.

```
fwconsole ma disable oracle_connector pms sangomaconnect sangomartapi adv_recovery
fwconsole ma disable restapps endpoint
fwconsole reload
```

### Install OSS Endpoint Manager for FreePBX 16

Commands to load the open source endpointman updated for FreePBX 16 comes from a [voip-info.org forums](https://www.voip-info.org/forum) thread.

As of April 2023, `endpointman` has `This branch is 3 commits ahead of FreePBX-ContributedModules:release/14.0.`

The process written below comes from Ward Mundy of [Nerd Vittles](https://nerdvittles.com/) in the [same voip-info.org thread as the fork](https://www.voip-info.org/forum/threads/oss-epm-for-freepbx-16-ipbx-2027.26880/page-2#post-169285).

#### In the command line

```
cd /var/www/html/admin/modules
wget http://incrediblepbx.com/ossepm16.tgz
tar zxvf ossepm16.tgz
rm -f ossepm16.tgz
rm -f /tmp/*
fwconsole ma install endpointman
fwconsole reload
```

#### In the UI

- Once installed, open FreePBX GUI and navigate to **Settings -> OSS Endpoint Manager**.
- Click on **Settings** and enter `http://provision.lol/` for **Package Server**. *(this is done on the older version)*
- Click on the **Options icon** *(the floating list icon on the right)* and choose **Package Manager**. Once it opens, click **Check for Updates button**.
- Click on the **Install** buttons for the Device Brands you wish to enable.

### Author's Specific Procedures

We're going to enable the Cisco/Linksys SPA8000 management because that is the model ATA device hardware we have tested and deployed with FreePBX.

- Scroll down to the **Cisco/Linksys** heading and click **Install**.
- Click **Enable** for the **SPA8000** model of ATA we use.

### Usage

In the UI

**Applications -> Extensions**

#### Errors

Failing page for this module: `http://192.168.101.2/admin/config.php?display=epm_devices`


> Array and string offset access syntax with curly braces is deprecated.

Affected files:

```
Whoops\Exception\ErrorException 
/var/www/html/admin/modules/endpointman/includes/json.inc232
4
Whoops\Run handleError
/var/www/html/admin/modules/endpointman/includes/functions.inc36
3
 require_once
/var/www/html/admin/modules/endpointman/includes/functions.inc36
2
endpointmanager __construct
/var/www/html/admin/modules/endpointman/functions.inc.php89
1
 endpointman_configpageinit
/var/www/html/admin/libraries/BMO/GuiHooks.class.php267
0
FreePBX\GuiHooks doConfigPageInits
/var/www/html/admin/config.php445
```

### Update Procedure

[Github commit diff for Bill Simon's fork](https://github.com/billsimon/endpointman/commit/ce96e6d3bbf41cf3d42f2bb76ef896b41352877d)

Before that, three commits ago is the actual "release for FreePBX 16"

https://github.com/billsimon/endpointman/commit/9b3a807970a983a5be35170029839bb3142444cc

Updating this in-place can be done by downloading the zip file from github and swapping in the directory.

```
cd /var/www/html/admin/modules/
rm -rf endpointman
wget https://github.com/billsimon/endpointman/archive/refs/heads/release/16.0.zip
unzip 16.0.zip
mv endpointman-release-16.0 endpointman
chown -R asterisk:asterisk endpointman
fwconsole reload
```

After this point the code changes discussed in this fork are required. As well as testing and probably revisions.


## Readme from Origin

```
 ______             _____  ______   __
|  ____|           |  __ \|  _ \ \ / /
| |__ _ __ ___  ___| |__) | |_) \ V /
|  __| '__/ _ \/ _ \  ___/|  _ < > <
| |  | | |  __/  __/ |    | |_) / . \
|_|  |_|  \___|\___|_|    |____/_/ \_\
Your Open Source Asterisk PBX GUI Solution
```
### What?
endpointman
This is a module for [FreePBX©](http://www.freepbx.org/ "FreePBX Home Page"). [FreePBX](http://www.freepbx.org/ "FreePBX Home Page") is an open source GUI (graphical user interface) that controls and manages [Asterisk©](http://www.asterisk.org/ "Asterisk Home Page") (PBX). FreePBX is licensed under GPL.
[FreePBX](http://www.freepbx.org/ "FreePBX Home Page") is a completely modular GUI for Asterisk written in PHP and Javascript. Meaning you can easily write any module you can think of and distribute it free of cost to your clients so that they can take advantage of beneficial features in [Asterisk](http://www.asterisk.org/ "Asterisk Home Page")

### Setting up a FreePBX system
[See our WIKI](http://wiki.freepbx.org/display/FOP/Install+FreePBX)
### License
[This modules code is licensed as GPLv3+](http://www.gnu.org/licenses/gpl-3.0.txt)
### Contributing
To contribute code or modules back into the [FreePBX](http://www.freepbx.org/ "FreePBX Home Page") ecosystem you must fully read our Code License Agreement. We are not able to look at or accept patches or code of any kind until this document is filled out. Please take a look at [http://wiki.freepbx.org/display/DC/Code+License+Agreement](http://wiki.freepbx.org/display/DC/Code+License+Agreement) for more information
### Issues
Please file bug reports at http://issues.freepbx.org# endpointman
