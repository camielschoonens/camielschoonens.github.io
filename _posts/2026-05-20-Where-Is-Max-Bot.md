---
layout: post
title: Tracking Max Verstappen's private jet with Home Assistant and ADS-B
date: 2026-05-19 10:00 +0200
description: How I built a Home Assistant automation that tracks Max Verstappen's private jet using ADS-B Exchange, reverse geocodes its position with Nominatim, and posts takeoff and landing updates to a dedicated Mastodon bot.
categories: [Home Assistant]
tags: [home-assistant, mastodon, ads-b, automation, aviation]
pin: false
toc: true
published: true
---

Every commercial and private aircraft broadcasting ADS-B is publicly visible to anyone willing to poll the right API. Max Verstappen's Dassault Falcon 900 with registration PH-UTL (Unleash The Lion) is no exception. I thought it would be fun to wire this into Home Assistant and have a dedicated Mastodon bot post updates whenever the jet takes off or lands somewhere. This idea is based of a Twitter Bot that does the sane.

The result is [@waarismax](https://social.schoonens.nl/@waarismax) on my own Fediverse instance. It posts publicly whenever PH-UTL moves. The Bot posts in Dutch.

## Why

Because I could.

## Finding the data

[adsb.lol](https://api.adsb.lol) is a free, no-auth API that queries live ADS-B data by ICAO hex code. A single GET request returns the current position and state of any broadcasting aircraft:

```
https://api.adsb.lol/v2/icao/4867e6
```

When the aircraft is airborne, the response includes an `ac` array with fields for altitude, ground speed, lat/lon, callsign, and more. When it's on the ground with its transponder off, `ac` is empty and `total` is `0`. That on/off toggle is the trigger for the automations.

## The sensor setup

Everything lives in `configuration.yaml`. The `rest:` block polls the API every 60 seconds and derives six sensors from the response:

{% raw %}
```
rest:
  - resource: https://api.adsb.lol/v2/icao/4867e6
	scan_interval: 60
	sensor:
	  - name: max_jet_status
		value_template: >
		  {% if value_json.total != 0 %}airborne{% else %}offline{% endif %}
	  - name: max_jet_altitude
		value_template: {{ value_json.ac[0].alt_baro | default('ground') }}
	  - name: max_jet_speed
		value_template: "{{ value_json.ac[0].gs | default(0) }}"
	  - name: max_jet_lat
		value_template: "{{ value_json.ac[0].lat | default('') }}"
	  - name: max_jet_lon
		value_template: "{{ value_json.ac[0].lon | default('') }}"
	  - name: max_jet_flight
		value_template: "{{ value_json.ac[0].flight | default('unknown') }}"
```
{% endraw %}

The `max_jet_status` sensor is the key one — it flips between `airborne` and `offline` and is what the takeoff and landing automations watch.

## Reverse geocoding the position

Lat/lon coordinates aren't readable in a Mastodon post. I wanted to read "Keulen, Duitsland" rather than "50.868, 7.121". OpenStreetMap's [Nominatim](https://nominatim.openstreetmap.org) API handles this for free, with no API key required, as long as you respect the fair use policy and don't hammer the API.

The approach I landed on (pun intended) avoids a fixed polling interval entirely. Instead of querying Nominatim every N seconds, a `rest_command` calls it on demand, and a dedicated automation fires that command whenever `sensor.max_jet_lat` changes state. This means Nominatim only gets called when the aircraft is actually moving.

{% raw %}
```
rest_command:
  update_max_jet_location:
	url: >
	  https://nominatim.openstreetmap.org/reverse?lat={{ states('sensor.max_jet_lat') }}&lon={{ states('sensor.max_jet_lon') }}&format=json&accept-language=nl
	method: GET
	headers:
	  User-Agent: "HomeAssistant/waarismax"
```
{% endraw %}

The result is parsed in the automation and written to an `input_text` helper:

{% raw %}
```
alias: '[Luchtvaart] Locatie bijwerken bij beweging'
triggers:
  - trigger: state
	entity_id: sensor.max_jet_lat
conditions:
  - condition: not
	conditions:
	  - condition: state
		entity_id: sensor.max_jet_lat
		state: ''
actions:
  - action: rest_command.update_max_jet_location
	response_variable: nominatim_response
  - action: input_text.set_value
	target:
	  entity_id: input_text.max_jet_location
	data:
	  value: >
		{{ nominatim_response['content']['address']['city'] |
		default(nominatim_response['content']['address']['town'] |
		default(nominatim_response['content']['address']['village'] |
		default('onbekend'))) }},
		{{ nominatim_response['content']['address']['country'] | default('') }}
```
{% endraw %}

The `city → town → village` fallback chain handles the fact that Nominatim doesn't always return a `city` field, depending on how rural the aircraft's position is.

## The Mastodon bot

I created a separate Mastodon account, `@waarismax@social.schoonens.nl`, on my self-hosted instance and I connected it to Home Assistant.

## The automations

Two automations watch `sensor.max_jet_status` and post to the bot when it changes. The `for:` duration debounces brief signal dropouts. I'm working with a 5-minute window for takeoff and landing right now. If needed I will optimize this further.

**Takeoff:**

{% raw %}
```
alias: '[Luchtvaart] Takeoff gedetecteerd'
triggers:
  - trigger: state
	entity_id: sensor.max_jet_status
	from: "offline"
	to: "airborne"
	for:
	  minutes: 5
actions:
  - action: mastodon.post
	data:
	  config_entry_id: YOUR_CONFIG_ENTRY_ID
	  visibility: public
	  status: |-
		🛬 Max Verstappen's vliegtuig is geland in {{ states('input_text.max_jet_location') }}.
	  language: nl
mode: single
```
{% endraw %}

**Landing:**

{% raw %}
```
alias: '[Luchtvaart] Landing gedetecteerd'
triggers:
  - trigger: state
	entity_id: sensor.max_jet_status
	from: "airborne"
	to: "offline"
	for:
	  minutes: 5
actions:
  - action: mastodon.post
	data:
	  config_entry_id: YOUR_CONFIG_ENTRY_ID
	  visibility: public
	  status: |-
		🛫 Max Verstappen's vliegtuig is opgestegen vanuit {{ states('input_text.max_jet_location') }}.
	  language: nl
mode: single
```
{% endraw %}

## What it looks like

A typical takeoff post from `@waarismax`:

![](/assets/img/SCR-20260520-hjji.png)

The live link goes directly to ADS-B Exchange with the aircraft pre-selected, so anyone following the bot can pull up the full flight track with one tap.