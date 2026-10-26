# Victron ESS Node-RED Manager – user guide

A Node-RED flow for a Victron GX device (Cerbo GX, Ekrano...) with ESS. It controls a PV +
battery system according to Czech spot electricity prices: it sells to the grid when it pays off,
charges from the grid when it is cheap, limits feed-in at high grid voltage, switches relays and
Shelly devices and calibrates the battery SOC. Everything is set up in a web dashboard
(Dashboard 2.0); an optional read-only mobile app is available.

This is the English edition of the flow (flow-en.json). The logic is identical to the Czech one;
only the dashboard texts are translated. Detailed technical documentation (how and why every
part works) exists in Czech only (cs/DOKUMENTACE.txt in the repository).

Made by Elektrárny Černý (https://elektrarnycerny.cz). Free for non-commercial use under the
PolyForm Noncommercial License 1.0.0; commercial use by agreement: pavel@elektrarnycerny.cz.


## 1. What it does

- Selling to the grid
  - (A) Selling at a high price: PV surplus whenever the sell price is above "Min. sell price for
    surplus"; energy from the battery above a set SOC, but only in the most expensive
    quarter of the day.
  - (B) Planned selling from the battery: every day a chosen share of the battery capacity is
    sold in the most expensive quarter-hours of the day (never below the price floor, the
    household reserve or the minimum SOC).
  - (C) Morning charging deferral (summer): when the morning sell price is higher than at noon,
    the battery is not charged in the morning, PV goes to the grid and the noon sun tops the
    battery up later. Has a weather brake (cloudy day = off).
  - "Full battery": above the set SOC the full allowed feed-in is opened, so a full battery does
    not throttle the PV.
- Charging from the grid
  - Minimum SOC by a set time - always, in the cheapest quarter-hours before the deadline.
  - Target SOC (winter) - only in quarter-hours below a price cap, reduced by the PV forecast.
  - Charging to 100 % at a negative final buy price.
- Summer and winter profiles - each season has its own settings; switching by date or manually.
- Prices: 15-minute spot prices of the selected market zone - Czechia (spotovaelektrina.cz, backup
  source OTE converted with the ČNB EUR rate, CZK) or another European day-ahead zone (Energy-Charts,
  EUR); the local time of the whole flow follows the zone, fixed prices as an option and as a fallback, distribution tariff HT/LT with a weekly
  schedule and buy/sell fees.
- Battery SOC calibration: the battery is regularly charged to 100 % - preferably from the sun;
  after the interval selling stops and the battery is topped up from the grid in the afternoon
  only by the small missing part.
- Grid voltage protection: step-wise reduction of feed-in at high phase voltage.
- Relays and Shelly: 2 GX relays, 2 local Shelly (Gen1 REST by IP), 2 Shelly Cloud devices -
  off / automatic (by SOC, PV power or surplus, time window) / always on.
- Dashboard: Overview (battery, energy flow, today's energy, plan, prices), settings with help
  (the (i) icon) for every field, statistics, configuration backup and restore.
- Mobile app (PWA): read-only live view, paired by a QR code.


## 2. Requirements

- Victron GX with Venus OS Large (Node-RED included) and ESS configured.
- MQTT enabled on the GX: Settings -> Services -> MQTT -> "MQTT on LAN (plaintext)".
- Dashboard 2.0: package `@flowfuse/node-red-dashboard` (tested with 1.33).
- Internet for spot prices and the weather forecast (Open-Meteo).
- For "Never discharge the battery to the grid" and PV surplus export: ESS setting
  "Feed-in excess DC-coupled PV" enabled.
- For the morning deferral: DVCC enabled on the GX (it limits the charging current).
- GX relays used by the flow must be set to "Manual" on the GX (Settings -> Relay).


## 3. Installation

1. On the GX (or via VRM -> Remote Console): enable MQTT (see above), write down the VRM Portal
   ID (Settings -> VRM online portal), set the relays to "Manual".
2. Open the Node-RED editor: https://<GX-address>:1881 (HTTPS and port 1881; the certificate is
   self-signed, so accept the browser warning).
3. Menu -> Manage palette -> Install -> `@flowfuse/node-red-dashboard`. Do not confuse it with
   `node-red-dashboard-2` or other similarly named packages.
4. Menu -> Import -> select flow-en.json -> Deploy.
5. Open the dashboard: https://<GX-address>:1881/dashboard -> "Installation setup" and fill in:
   VRM Portal ID first (without it nothing is written to the Victron; if the flow sees it on MQTT,
   it is filled in automatically), installation name, battery capacity (kWh), PV size (kWp),
   coordinates (for the weather forecast), nominal max. grid feed-in and max. charging power from
   the grid. The "Details check" row shows what is still missing.
6. Overview -> "Connection to Victron (MQTT)" should show "OK" within about 15 s.
7. "Tariffs and prices": first the market (zone) - prices, currency and local time follow it; when
   the currency changes, the price settings are converted at 25 CZK/EUR, so check them. Then spot or
   fixed buy/sell price, buy fees HT/LT, selling fee, weekly
   low-tariff schedule.
8. "Modes": leave "Operating mode" on Manual until you have checked everything. In Manual the
   flow calculates and shows its recommendation ("MANUAL, recommended: ...") but writes nothing
   (except battery calibration). Then switch to an automatic mode.

Safe first test in Manual: on the Modes page set a small "Import from the grid" value in the Manual control block (e.g. 100 W) and
check on the GX/VRM that the grid setpoint changed - positive = import from the grid, negative =
export.


## 4. Dashboard pages

Overview
  Battery (SOC, power, calibration state), energy flow (PV, consumption, surplus, grid), today's
  energy (exported, imported, PV production - from the Victron counters, same as VRM), battery
  selling plan and grid charging plan (two rows), regulation state, prices, tariff, forecast, morning
  deferral and connection state. Every row has a help text under the (i) icon.

Installation setup
  Installation details, grid feed-in limit (changes need confirmation), charging power, battery
  SOC calibration settings, mobile app pairing, full configuration with copy / download / restore
  from backup (with a preview of changes; the VRM Portal ID is never taken over; the previous
  state can be reverted).

Tariffs and prices
  Price types, fixed prices, fees, spot price chart for today/tomorrow, weekly low-tariff windows.

Modes
  Operating mode (automatic by date / summer / winter, or Manual), idle setpoints for summer and
  winter, winter date range, and the settings of the summer and winter profiles: selling to the
  grid and from the battery (reserve, min. sell price, mechanism A threshold, mechanism B volume
  and price floor, full battery threshold, never discharge), charging from the grid (minimum SOC
  and its deadline, negative price; winter also target SOC, deadline and price cap) and the
  morning deferral (summer). Only one profile is shown at a time ("Show / edit settings for").
  In Manual only the "Manual control" block is shown.
  Field names are written in plain language (e.g. "Battery reserve for the house only", "Min.
  sell price for surplus"); the technical names (SOC, setpoint,
  mechanism A/B/C) are explained in the (i) help texts.

Relays and Shelly
  Mode, thresholds and time windows of the GX relays and Shelly devices, and their current states.

Voltage and protection
  Phase voltages, protection state and allowed power cap, protection settings, worst voltage chart.

Statistics
  Export to the grid and device switching for today and yesterday.

README / Changelog
  This guide and the list of changes.


## 5. Key concepts

- Sell price = spot (or fixed) price minus the selling fee. Buy price = spot (or fixed) price plus
  the HT or LT fee according to the tariff schedule. All decisions use these final prices.
- Household reserve: no selling from the battery and no idle feed-in goes below it.
- Minimum SOC: charged from the grid by its deadline at any price; also the floor for selling.
- Idle setpoint: the grid setpoint held when no mechanism acts (0 = zero at the meter; negative =
  slight feed-in; positive = slight import).
- Regulation reasons shown in "Export/charging state": no-action / idle-setpoint, sell-A-surplus,
  sell-A-from-battery, sell-from-battery (B), sell-A-full-battery, A-waiting-for-surplus,
  morning-deferral, grid-charging, calibration-day / calibration-waiting / calibration-solar,
  calibration-topup, pv-surplus-only.


## 6. Battery SOC calibration

The BMS needs to reach 100 % now and then to correct its SOC estimate.
1. In the last 4 days before the interval ends (days 10-13 for 14 days) a clear day with enough
   forecast production becomes a solar calibration day: no selling, the battery charges from the
   sun only.
2. When the interval has elapsed, calibration mode starts: no selling from the battery, idle
   setpoint 0, PV surplus to the battery. From the set time (default 13:00) the battery is topped
   up from the grid to 100 % as soon as it exceeds the set SOC (default 95 %).
3. Safety net: if it never gets there, the day with the best forecast among the 3 days after the
   interval is used and the battery is topped up at any SOC in the afternoon.
The Overview shows a red "Battery calibration - selling off" / "Battery calibration day" next to
the battery. Settings and a detailed help are on the "Installation setup" page.


## 7. Updating to a new version

Delete all tabs of the flow in the Node-RED editor first (Cmd/Ctrl-click every tab header, then
Delete), import the new flow-en.json and Deploy. All settings are stored on the GX and are kept.
Do not use "Replace" on import - nodes removed in the new version would stay in the editor.


## 8. Data sent to the author's server

The flow reports to vrm.elektrarnycerny.cz:
- a heartbeat 4 times a day: VRM Portal ID, installation name, flow version, mode, season, battery
  capacity, kWp, selling settings, SOC, current regulation reason and the connection state,
- live data for the mobile app every 2 minutes: PV power, consumption, grid, battery, SOC,
  prices and the plan text; the server keeps only the last 24 hours.
No passwords, addresses or Shelly keys are sent. Installations of the public version are kept
separate from the author's own installations. To opt out, disable the nodes "POST heartbeat" and
"POST zive udaje" on the "Statistika" tab in the editor - the mobile app then stops working, the
regulation is not affected.


## 9. Mobile app

Installation setup -> Mobile app shows a QR code within 2 minutes after the VRM Portal ID is
filled in (the installation needs internet). Scan it with the phone, open the link and choose
"Add to Home Screen". The app only displays data. A new key can be generated at any time; it
invalidates all paired phones. The app is in English when the flow is the English
edition (before the first data arrive it follows the phone language).


## 10. License and support

PolyForm Noncommercial License 1.0.0 - free for personal and other non-commercial use including
modifications. Commercial use (e.g. installing for paying customers) by agreement:
pavel@elektrarnycerny.cz. Ideas and bug reports to the same address. If the flow serves you well,
a contribution to its development is welcome (details on the "Installation setup" page).
