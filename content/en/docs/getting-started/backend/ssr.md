---
title: "How-to publish storage site report"
linkTitle: "Storage Space Report"
weight: 4
noComment: true
description: >
  Learn how to allow VO’s authenticated users to access JSON report.
---

The following is an example of how to make the storage site report in JSON readble from all VOs' authenticated users through a WebDAV endpoint.

## Configure a storage area 

The aim is allowing all the users to access - through a WebDAV endpoint - the JSON storage report.

In the following example, a dedicated storage area named `info` has been added and configured to be readable-only by all the users.

### Puppet configuration

There's no need to add the `info` storage area to the ones managed by StoRM Backend. We can add the `info` storage area only to `storm::webdav` class in order to implicitly say that this storage area will be accessed only through HTTP and not through SRM. 
Configure this storage area with no VOs and set `authenticated_read_enabled` as true to allow browsing the report with your personal x509 certificate.

```puppet
class { 'storm::webdav':
  # ...
  storage_areas => [{
    'name'                       => 'info',
    'root_path'                  => '/storage/info',
    'access_points'              => ['/info'],
    'vos'                        => [],
    'authenticated_read_enabled' => true,
  },
  # ...
  ]
}
```

Apply Puppet configuration.

## Periodically reload report 

### Refresh script 

Write a simple bash script which runs info-provider, gets the updated json-report and replaces the old one.

```bash
#!/bin/bash
set -x
TMP_REPORT_PATH="/tmp/site-report-$(date +%s).json"
TARGET_REPORT_PATH="/storage/info/report.json"
if [ $# -gt 0 ]; then
    TARGET_REPORT_PATH=$1
fi
# refresh report
/usr/libexec/storm-info-provider get-report-json -o $TMP_REPORT_PATH
# copy report to storage area
cp $TMP_REPORT_PATH $TARGET_REPORT_PATH
chown storm:storm $TARGET_REPORT_PATH
```

Save it into `/root/update-report.sh`

### Cron-job

Create a cron-job that runs `update-report.sh` script hourly.

Create `/etc/cron.d/update-site-report` as follow:

```bash
*/30 * * * *      root    /bin/bash /root/update-report.sh
```

The `update-report.sh` script will be executed each 30 minutes.
Read more info about cron jobs into the [man page](https://crontab.guru/crontab.5.html).

### Configure periodic updates with Puppet

The same update script has been added to Puppet module since version 0.4.4.

You can add to your manifest (where StoRM Backend is defined) the following resources:

```puppet
file { '/root/update-site-report.sh':
  ensure => 'present',
  source => 'puppet:///modules/storm/update-site-report.sh',
}
cron { 'update-site-report':
  ensure  => 'present',
  command => '/bin/bash /root/update-site-report.sh',
  user    => 'root',
  minute  => '*/20',
  require => File['/root/update-site-report.sh'],
}
exec { 'create-site-report':
  command => '/bin/bash /root/update-site-report.sh',
  require => File['/root/update-site-report.sh'],
}
```

This snippet executes the script and also creates the cron job.
In alternative, you can use a defined type that does the same things:

```puppet
storm::backend::storage_site_report { 'storage-site-report':
  report_path => '/storage/info/report.json',
  minute      => '*/20',
}
```

The `report_path` is the internal `info` storage area path where report has to be created.
The `minute` can be used to customize the cron's minute parameter.

## Get JSON report through HTTP 

Acting as `test0` user of test.vo VO, we can access storage area with a simple http get.

We need `test0` user's certificate `test0.cert.pem` and his unencrypted key `test0.ukey.pem`.


```shell
$ curl https://omii005-vm03.cnaf.infn.it:8443/info/report.json \
    --cert ./test0.cert.pem \
    --key ./test0.ukey.pem \
    --capath /etc/grid-security/certificates
```

Output:

```json
{
    "storageservice": {
        "capabilities": [
            "data.management.transfer", 
            "data.management.storage"
        ], 
        "implementation": "storm", 
        "implementationversion": "1.11.16", 
        "latestupdate": 1571388242, 
        "name": "storm-testbed", 
        "qualitylevel": "pre-production", 
        "storageendpoints": [
            {
                "assignedshares": [
                    "all"
                ], 
                "capabilities": [
                    "data.management.transfer", 
                    "data.management.storage"
                ], 
                "endpointurl": "httpg://omii005-vm03.cnaf.infn.it:8444/srm/managerv2", 
                "interfacetype": "srm", 
                "interfaceversion": "2.2", 
                "name": "SRM_0", 
                "qualitylevel": "pre-production"
            }, 
            {
                "assignedshares": [
                    "all"
                ], 
                "capabilities": [
                    "data.management.transfer", 
                    "data.management.storage"
                ], 
                "endpointurl": "http://omii005-vm03.cnaf.infn.it:8085", 
                "interfacetype": "DAV", 
                "interfaceversion": "1.1", 
                "name": "HTTP_0", 
                "qualitylevel": "pre-production"
            }, 
            {
                "assignedshares": [
                    "all"
                ], 
                "capabilities": [
                    "data.management.transfer", 
                    "data.management.storage"
                ], 
                "endpointurl": "https://omii005-vm03.cnaf.infn.it:8443", 
                "interfacetype": "DAV", 
                "interfaceversion": "1.1", 
                "name": "HTTPS_0", 
                "qualitylevel": "pre-production"
            }
        ], 
        "storageshares": [
            {
                "accesslatency": "online", 
                "assignedendpoints": [
                    "all"
                ], 
                "name": "INFO_TOKEN", 
                "path": [
                    "/info"
                ], 
                "retentionpolicy": "replica", 
                "servingstate": "open", 
                "timestamp": 1571388242, 
                "totalsize": 1000000000, 
                "usedsize": 5824, 
                "vos": [
                    "test.vo", 
                    "test.vo.2"
                ]
            },
            {
                "accesslatency": "nearline", 
                "assignedendpoints": [
                    "all"
                ], 
                "name": "TAPE_TOKEN", 
                "path": [
                    "/tape"
                ], 
                "retentionpolicy": "custodial", 
                "servingstate": "open", 
                "timestamp": 1571388242, 
                "totalsize": 14000000000, 
                "usedsize": 464525, 
                "vos": [
                    "test.vo.2"
                ]
            },
            {
                "accesslatency": "online", 
                "assignedendpoints": [
                    "all"
                ], 
                "name": "TESTVO_TOKEN", 
                "path": [
                    "/test.vo"
                ], 
                "retentionpolicy": "replica", 
                "servingstate": "open", 
                "timestamp": 1571388242, 
                "totalsize": 24000000000, 
                "usedsize": 1315148151, 
                "vos": [
                    "test.vo"
                ]
            }, 
            {
                "accesslatency": "online", 
                "assignedendpoints": [
                    "all"
                ], 
                "name": "OAUTHAUTHZ_TOKEN", 
                "path": [
                    "/oauth-authz"
                ], 
                "retentionpolicy": "replica", 
                "servingstate": "open", 
                "timestamp": 1571388242, 
                "totalsize": 4000000000, 
                "usedsize": 4096, 
                "vos": [
                    "test.vo"
                ]
            }
        ]
    }
}
```

