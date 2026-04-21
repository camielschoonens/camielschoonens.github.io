---
layout: post
title: "Home Assistant Bot tracks planes flying over my house"
date: 2026-04-21 15:50
description: "I connected a Home Assistant Bot to Flight Radar on my self-hosted Fediverse (Mastodon) instance to receive private notifications about planes flying over my house."
categories: [Home Assistant]
tags: [home-assistant, mastodon, flightradar, automation]
pin: false
toc: false
published: true
---

I have a small [Home Assistant Bot](https://social.schoonens.nl/@homeassistant) running on my own Fediverse instance. It sends me private messages about what’s happening around the house. At some point I thought: why stop at lights and energy monitoring when I can do more fun things by connecting different APIs?

So I connected the bot to Flight Radar.

## What it does

Whenever a plane flies over my house on its approach to Amsterdam Schiphol, the bot sends me a (Dutch) private message on Mastodon with all the details:

- ✈️ The airline and flight number  
- 🛫 Origin and destination  
- 📏 The altitude at which it’s passing  
- 🕐 The expected arrival time at Schiphol

A typical message looks like this:

> 🤖 KLM van 🛫 Singapore naar Amsterdam 🛬 vliegt op een hoogte van 1494 meter boven #Oudewater. Dit is een vliegtuig van het type Boeing 777-306(ER). De verwachte aankomsttijd van deze vlucht op Amsterdam Schiphol is 08:15.
{: .prompt-info }

## Why the Fediverse?

1. Because I could.  
2. I already run a Mastodon instance for our household, so the Fediverse was a natural fit. The bot posts directly to my account as a private/direct message, which means no extra apps, no push notification service to maintain — just a message popping up in my timeline like any other. It keeps everything self-hosted and under my control, which fits perfectly with the spirit of both Home Assistant and the Fediverse.