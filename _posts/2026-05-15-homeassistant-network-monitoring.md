---
layout: post
title: Building a network monitoring dashboard in Home Assistant
date: 2026-05-15 20:00 +0200
description: How I built a real-time network monitoring dashboard in Home Assistant using SNMP, derivative sensors, and different charts, covering WAN throughput, per-AP bandwidth, and daily/monthly data usage.
categories: [Home Assistant]
tags: [home-assistant, unifi, snmp, networking, dashboard]
pin: false
toc: true
published: true
---

I run a full UniFi stack at home, a Cloud Gateway Fiber Gateway (UCG), five different switches, and five access points are spread across the house. The UniFi integration for Home Assistant gives you device status and connected client counts, but it doesn't expose bandwidth data as sensors. I wanted to see real-time throughput on my HA dashboard. This post documents how I built that.

## Why SNMP

My first instinct was to look for a purpose-built integration, but nothing reliable existed for per-AP bandwidth at the time. The UniFi switches (USW Flex 2.5G) don't support SNMP at all, which ruled out the obvious approach of polling switchport counters. The gateway and access points do support SNMP though, which turned out to be enough.

SNMP is the right approach as every UniFi device keeps internal byte counters per interface. By polling those counters periodically, you can calculate throughput as a delta over time. The performance impact is negligible, a few hundred bytes per query over UDP, even at 10-second intervals.

## Setting up SNMP

SNMP is disabled by default on the UCG. Enable it under **Settings → System → SNMP** in the UniFi console, set version to SNMP v2c, and note your community string.

Before adding any sensors to Home Assistant, I used `snmpwalk` from my Mac to identify the right interface:

```bash
snmpwalk -v2c -c homelab 192.168.1.1 1.3.6.1.2.1.2.2.1.2
```

This returns all interfaces the device knows about. For my UCG, the WAN uplink is `eth6.701`. VLAN 701 is what my ISP uses. The interface index was 21. I confirmed it had real traffic with:

```bash
snmpget -v2c -c homelab 192.168.1.1 1.3.6.1.2.1.31.1.1.1.6.21 1.3.6.1.2.1.31.1.1.1.10.21
```
For the access points, `eth0` is the wired uplink on all U7 Pro APs (index 3). The older AC Mesh in the garage has a different interface layout — `eth0` sits at index 2 there.

## The sensor pipeline

Getting from raw SNMP data to a Mbit/s reading in HA takes three layers:

**1. SNMP sensors** — raw cumulative byte counters, tagged `state_class: total_increasing` so HA handles resets correctly:

```yaml
- platform: snmp
  host: 192.168.1.1
  community: homelab
  version: "2c"
  scan_interval: 10
  name: "WAN In Octets"
  baseoid: 1.3.6.1.2.1.31.1.1.1.6.21
  unit_of_measurement: "octets"
  state_class: total_increasing
```

**2. Derivative sensors** — calculate the rate of change in octets per second:

```yaml
- platform: derivative
  source: sensor.wan_in_octets
  name: "WAN Download Octets per Sec"
  unique_id: wan_download_octets_per_sec
  unit_time: s
  time_window: "00:00:30"
  round: 2
```

The `time_window` of 30 seconds smooths out polling jitter without introducing too much lag.

**3. Template sensors** — convert octets/s to Mbit/s (octets × 8 / 1,000,000):

```yaml
- name: "WAN Download"
  unique_id: wan_download_mbps
  unit_of_measurement: "Mbit/s"
  state_class: measurement
  state: "{{ [((states('sensor.wan_download_octets_per_sec') | float(0)) * 8 / 1000000) | round(2), 0] | max }}"
```

The `| max` with 0 prevents negative spikes when the derivative briefly dips below zero.

I went through a few failed approaches before landing on this pipeline. My first attempt used template sensors that stored the previous value as an attribute and calculated the delta themselves. This fails because HA evaluates `current` and `previous_value` in the same render cycle, so the delta is always zero. The `derivative` platform solves this cleanly by doing the math in HA's core rather than in a template.

## The dashboard

For the WAN graph I used [mini-graph-card](https://github.com/kalkih/mini-graph-card). It's cleaner than apexcharts for simple time-series and doesn't need a formatter function to display units:

```yaml
type: custom:mini-graph-card
name: WAN Bandbreedte
hours_to_show: 1
points_per_hour: 120
update_interval: 10
line_width: 2
smooth: false
show:
  labels: true
  points: false
  legend: true
  fill: fade
  units: false
entities:
  - entity: sensor.wan_download
    name: Download
    color: '#00b4d8'
  - entity: sensor.wan_upload
    name: Upload
    color: '#f77f00'
```

For the five access points, I grouped them all together on the right side of the dashboard. Each AP gets a distinct color pair for download and upload.

![](/assets/img/SCR-20260515-ruhn.png)

## Data usage tracking

Once the SNMP sensors were in place, adding daily and monthly data counters was straightforward using `utility_meter`:

```yaml
utility_meter:
  wan_download_daily:
    source: sensor.wan_in_octets
    name: WAN Download - Dag
    cycle: daily
  wan_download_monthly:
    source: sensor.wan_in_octets
    name: WAN Download - Maand
    cycle: monthly
```

I added template sensors on top to convert octets to GB for display, since raw octet values in the billions aren't useful on a dashboard card.

## A note on entity IDs

One thing that caused me grief: when migrating from the legacy `platform: template` syntax to the modern `template:` block syntax (required since HA 2026.6), entity IDs change. The legacy syntax used the `friendly_name` key and generated entity IDs from the sensor key name. The modern syntax generates entity IDs from the `name` field. If you have utility meters, automations, or other sensors referencing the old entity IDs, they all break silently.

The fix: always set `unique_id` on every sensor. HA uses the unique_id as the stable internal identifier, so renaming via the UI won't break references. The entity_id is just the default label HA generates from the name. You can override it in the UI once the sensor exists.
