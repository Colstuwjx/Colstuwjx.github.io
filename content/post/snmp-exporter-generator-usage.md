---
title: "Easy your life by using the snmp-exporter generator"
date: 2017-07-01T20:00:00+08:00
tags: ["snmp-exporter", "prometheus", "generator"]
categories: ["monitoring"]
comments: false
showMeta: false
showAction: false
aliases:
    - /snmp-exporter-generator-tutorial/
---

This is a post for explaining how to use snmp exporter generator...

<!--more-->

[Prometheus](https://github.com/prometheus) is a great monitoring system, and our devops engineer team has been using it in production for months. When it came to network monitoring, we choose to use the [snmp-exporter](https://github.com/prometheus/snmp_exporter) to collect the metrics from the network switches.

snmp-exporter is also a great tool to do such snmp walk and collect the metrics we need. We could also take this a step further, make our life much easier by using the snmp-exporter [generator](https://github.com/prometheus/snmp_exporter/tree/master/generator).Yes, that is what I want to share in this article.

### MIB & OID

> TIPS: you can skip this section if you already know the SNMP protocol, and understand the meaning of the MIB and OID.

First of all, we should understand two terms in the snmp protocol —— MIB and OID.

#### MIB

[MIB](https://en.wikipedia.org/wiki/Management_information_base), a short for management information base, it is a database used for managing the entities in a communication network. The database is hierarchical (tree-structured) and each entry is addressed through an object identifier ([OID](https://en.wikipedia.org/wiki/Object_identifier)).

It is often used to refer to as MIB-module in the snmp monitoring. And in a MIB module, it usually has some `DEFINITIONS` at the beginning, said which vendor defined it, and the last update date, the dependency modules etc. The mainly content in the MIB module is the MIB object.

A MIB object (sometimes called a managed object, an object, or a MIB) is one of any number of specific characteristics of a managed device. Managed objects are made up of one or more object instances (identified by their OIDs), which are essentially variables.There are two types of managed objects exist:

* Scalar objects define a single object instance.
* Tabular objects define multiple related object instances that are grouped in MIB tables.

#### OID

We have mentioned the `OID` for many times at the before. An [OID](https://en.wikipedia.org/wiki/Object_identifier) corresponds to a node in the "OID tree" or MIB hierarchy, maybe you can just think about the IP address in the TCP/IP protocol.

Each OID node in the tree is represented by a series of integers separated by periods, corresponding to the path from the root through the series of ancestor nodes, to the node. Thus, an OID denoting Intel Corporation appears as follows,

`1.3.6.1.4.1.343`

and corresponds to the following path through the OID tree:

```
1 ISO
1.3 identified-organization,
1.3.6 dod,
1.3.6.1 internet,
1.3.6.1.4 private,
1.3.6.1.4.1 IANA enterprise numbers,
1.3.6.1.4.1.343 Intel Corporation
```

An OID node comes with a specified data type. The first version of the SMI (SMIv1) specifies the use of a number of SMI-specific data types, which are divided into two categories:

* Simple data types, such as integer, Octet strings etc;
* Application-wide data types, such as the Network addresses, Counters, Gauges etc.

Refer to the wiki: https://en.wikipedia.org/wiki/Object\_identifier and https://en.wikipedia.org/wiki/Management\_information\_base for the details.

#### MIB Object sample

The following is a short MIB object sample:

```
# MIB Object name is ifNumber
# and the data type is Integer32
# and it is read-only
# ...
ifNumber  OBJECT-TYPE
    SYNTAX      Integer32
    MAX-ACCESS  read-only
    STATUS      current
    DESCRIPTION
            "The number of network interfaces (regardless of their
            current state) present on this system."
    ::= { interfaces 1 }
```

By defining the MIB and OID, a MIB module announces the managed objects for the device, thus, we could do some admin works or monitor tasks towards the snmp-enabled devices via the snmp protocol.

### snmp.yml

> TIPS:
we could find out the `SNMP exporter Get Started` page from here: https://github.com/prometheus/snmp_exporter/blob/master/README.md

As the name said, `snmp-exporter` collect the metrics via snmp protocol. Thus, it needs the OIDs as its walk target. It's not easy to get the OIDs from the MIB module, so we need to find out the OIDs, and maybe the result is just like the following case:

```
# let's said that we need the metrics from the `hwCpuDevTable`.
# So I need to find out the OIDs for each item by manually.
# `hwCpuDevIndex`, `hwCpuDevDuty`, ..., it really makes my life much harder...
# we won't cover the detail configs below, 
# refer to the snmp-exporter official repository for details: https://github.com/prometheus/snmp_exporter
huawei:
  walk:
  - 1.3.6.1.4.1.2011.6.3.4
  metrics:
  - name: hwCpuDevIndex
    oid: 1.3.6.1.4.1.2011.6.3.4.1.1
    type: gauge
    indexes:
    - labelname: hwFrameIndex
      type: gauge
    - labelname: hwSlotIndex
      type: gauge
    - labelname: hwCpuDevIndex
      type: gauge
  - name: hwCpuDevDuty
    oid: 1.3.6.1.4.1.2011.6.3.4.1.2
    type: gauge
    indexes:
    - labelname: hwFrameIndex
      type: gauge
    - labelname: hwSlotIndex
      type: gauge
    - labelname: hwCpuDevIndex
      type: gauge
  - name: hwAvgDuty1min
    oid: 1.3.6.1.4.1.2011.6.3.4.1.3
    type: gauge
    indexes:
    - labelname: hwFrameIndex
      type: gauge
    - labelname: hwSlotIndex
      type: gauge
    - labelname: hwCpuDevIndex
      type: gauge
  - name: hwAvgDuty5min
    oid: 1.3.6.1.4.1.2011.6.3.4.1.4
    type: gauge
    indexes:
    - labelname: hwFrameIndex
      type: gauge
    - labelname: hwSlotIndex
      type: gauge
    - labelname: hwCpuDevIndex
      type: gauge
```

We got the `snmp.yml` right now, and we could use it as the snmp-exporter config, start to collect the `hwCpuDevDuty`, `hwAvgDuty1min`,... metrics into the prometheus.

Wait... we still need to find out the OIDs from the MIB module by manually, is there any easier way?

### snmp-exporter generator

Of Course! Prometheus team built an amazing [generator](https://github.com/prometheus/snmp_exporter/tree/master/generator) to easy our life!

SNMP generator needs a configuration file `generator.yml`, and uses NetSNMP to parse MIBs, and then generates `snmp.yml` config for the snmp_exporter using them.

#### build it

Firstly, we need to build the `generator` binary:

```
sudo apt-get install build-essential libsnmp-dev snmp-mibs-downloader  # Debian-based distros
go get github.com/prometheus/snmp_exporter/generator
cd ${GOPATH-$HOME/go}/src/github.com/prometheus/snmp_exporter/generator
go build
```

#### preparing MIBs

We must prepare some MIB modules which indicates the metrics we need. In our case, it was `hwCpuDevTable`, and we could find the MIB module [here](https://github.com/librenms/librenms/blob/master/mibs/huawei/HUAWEI-CPU#L26).

> TIPS: we could find the MIBs in many ways, one of them is that refering to https://github.com/librenms/librenms/blob/master/mibs

As the MIB shown, `hwCpuDevTable` has some imports, they're:

* `hwDev` from `HUAWEI-MIB`;
* `hwFrameIndex`, `hwSlotIndex` from `HUAWEI-DEVICE-MIB`;
*  `OBJECT-GROUP`, `MODULE-COMPLIANCE` from `SNMPv2-CONF`;
*  `Gauge`, `OBJECT-TYPE`, `MODULE-IDENTITY` from `SNMPv2-SMI`.

Dependencies could be found in [HUAWEI-MIB](https://github.com/librenms/librenms/blob/master/mibs/huawei/HUAWEI-MIB), [HUAWEI-DEVICE-MIB](https://github.com/librenms/librenms/blob/master/mibs/huawei/HUAWEI-DEVICE), [IF-MIB](https://github.com/librenms/librenms/blob/master/mibs/IF-MIB)(as SNMPv2-SMI, SNMPv2-CONF is from that). BTW, don't forget the `HUAWEI-CPU` MIB itself.

Put the `HUAWEI-MIB`, `HUAWEI-DEVICE-MIB`, `IF-MIB`, `HUAWEI-CPU` in a location NetSNMP can read them from. `$HOME/.snmp/mibs` is one option.

#### preparing generator.yml

As we need to scrape `hwCpuDevTable`, our `generator.yml` would be:

```
modules:
# we need to generate hwCpuDevTable metrics.
# and the module name in the `snmp.yml` will be `huawei`.
  huawei:
    walk: [hwCpuDevTable]
```

#### generate it!

Put the configured `generator.yml` with the `generator` binary together, and run `generator` with the following instruction:

```
./generator generate
```

The generator reads in from `generator.yml` and writes to `snmp.yml`. In our case, the output `snmp.yml` would be:

```
huawei:
  walk:
  - 1.3.6.1.4.1.2011.6.3.4
  metrics:
  - name: hwCpuDevIndex
    oid: 1.3.6.1.4.1.2011.6.3.4.1.1
    type: gauge
    indexes:
    - labelname: hwFrameIndex
      type: gauge
    - labelname: hwSlotIndex
      type: gauge
    - labelname: hwCpuDevIndex
      type: gauge
  - name: hwCpuDevDuty
    oid: 1.3.6.1.4.1.2011.6.3.4.1.2
    type: gauge
    indexes:
    - labelname: hwFrameIndex
      type: gauge
    - labelname: hwSlotIndex
      type: gauge
    - labelname: hwCpuDevIndex
      type: gauge
  - name: hwAvgDuty1min
    oid: 1.3.6.1.4.1.2011.6.3.4.1.3
    type: gauge
    indexes:
    - labelname: hwFrameIndex
      type: gauge
    - labelname: hwSlotIndex
      type: gauge
    - labelname: hwCpuDevIndex
      type: gauge
  - name: hwAvgDuty5min
    oid: 1.3.6.1.4.1.2011.6.3.4.1.4
    type: gauge
    indexes:
    - labelname: hwFrameIndex
      type: gauge
    - labelname: hwSlotIndex
      type: gauge
    - labelname: hwCpuDevIndex
      type: gauge
```

Yes, we got the `snmp.yml`, same with the previous one, but it is automatically generated by snmp-exporter `generator` at this time!

### happy ending

Aha, our life becomes much easier by using the snmp-exporter generator to automatically generate the snmp-exporter configs, happy to see it!

### references

* https://en.wikipedia.org/wiki/Management\_information\_base
* https://en.wikipedia.org/wiki/Object_identifier
* https://github.com/librenms/librenms/tree/master/mibs
* http://net-snmp.sourceforge.net/dev/agent/parse\_8h_source.html
* https://github.com/prometheus/snmp_exporter
* https://github.com/prometheus/snmp_exporter/pull/158

```
Enjoy!
Colstuwjx
2017.07.01
```
