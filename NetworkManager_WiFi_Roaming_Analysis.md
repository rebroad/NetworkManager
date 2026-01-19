# NetworkManager WiFi Roaming and BSSID Selection Analysis

## Overview

This document provides a detailed analysis of how NetworkManager decides when to roam from one BSSID to another and how it chooses which BSSID to connect to initially. The analysis is based on examining the source code of NetworkManager (~/src/NetworkManager) and wpa_supplicant (~/src/hostap/wpa_supplicant).

## 1. Initial BSSID Selection

### NetworkManager's Role

NetworkManager uses `nm_wifi_aps_find_first_compatible()` function to find the first compatible access point (AP) in the scan results list. This function iterates through a linked list of APs maintained by NetworkManager and returns the first one that matches the connection criteria.

Key points:
- APs are linked using `c_list_link_tail()`, meaning they are added to the end of the list as they are discovered
- The list is not sorted by any specific criteria (signal strength, frequency, etc.)
- The function simply returns the first compatible AP in the order they were discovered

### Code Reference

```c
// src/core/devices/wifi/nm-wifi-ap.c
NMWifiAP *
nm_wifi_aps_find_first_compatible(const CList *aps_lst_head, NMConnection *connection)
{
    NMWifiAP *ap;

    g_return_val_if_fail(connection, NULL);

    c_list_for_each_entry (ap, aps_lst_head, aps_lst) {
        if (nm_wifi_ap_check_compatible(ap, connection))
            return ap;
    }
    return NULL;
}
```

### Compatibility Check

The compatibility check (`nm_wifi_ap_check_compatible()`) verifies several criteria:
- SSID match
- BSSID match (if specified in the connection profile)
- Mode match (infrastructure, adhoc, mesh, etc.)
- Band match (if specified in the connection profile)
- Channel match (if specified in the connection profile)
- Security compatibility

## 2. Roaming Decision Logic

### Background: wpa_supplicant Integration

NetworkManager primarily relies on wpa_supplicant to make roaming decisions. It configures wpa_supplicant with a "bgscan" (background scan) parameter that controls when scans for roaming candidates are performed.

### NetworkManager's Roaming Configuration

NetworkManager configures bgscan in `nm_supplicant_config_add_bgscan()`:

```c
// src/core/supplicant/nm-supplicant-config.c
gboolean
nm_supplicant_config_add_bgscan(NMSupplicantConfig *self,
                                NMConnection       *connection,
                                guint               num_seen_bssids,
                                GError            **error)
{
    NMSettingWireless         *s_wifi;
    NMSettingWirelessSecurity *s_wsec;
    const char                *bgscan;

    s_wifi = nm_connection_get_setting_wireless(connection);
    g_assert(s_wifi);

    if (NM_IN_STRSET(nm_setting_wireless_get_mode(s_wifi),
                     NM_SETTING_WIRELESS_MODE_AP,
                     NM_SETTING_WIRELESS_MODE_ADHOC))
        return TRUE;

    if (nm_setting_wireless_get_bssid(s_wifi))
        return TRUE;

    bgscan = "simple:30:-70:86400";

    if (num_seen_bssids > 1u
        || ((s_wsec = nm_connection_get_setting_wireless_security(connection))
            && NM_IN_STRSET(nm_setting_wireless_security_get_key_mgmt(s_wsec),
                            "ieee8021x",
                            "wpa-eap",
                            "wpa-eap-suite-b-192")))
        bgscan = "simple:30:-65:300";

    return nm_supplicant_config_add_option(self, "bgscan", bgscan, -1, FALSE, error);
}
```

### Bgscan Configuration Meaning

NetworkManager uses the "simple" bgscan algorithm with two main configurations:

1. **Default (PSK/WPA-Personal)**: `simple:30:-70:86400`
   - Scan every 30 seconds if signal strength < -70 dBm
   - Scan every 86400 seconds (24 hours) if signal strength > -70 dBm

2. **Enterprise (WPA-EAP/802.1x) or Multiple APs**: `simple:30:-65:300`
   - Scan every 30 seconds if signal strength < -65 dBm
   - Scan every 300 seconds (5 minutes) if signal strength > -65 dBm

### Wpa_supplicant's Role in Roaming

Wpa_supplicant implements the actual roam decision logic in `wpa_supplicant_need_to_roam_within_ess()`:

```c
// src/hostap/wpa_supplicant/events.c
int wpa_supplicant_need_to_roam_within_ess(struct wpa_supplicant *wpa_s,
                                           struct wpa_bss *current_bss,
                                           struct wpa_bss *selected,
                                           bool poll_current)
{
    int min_diff, diff;
    int cur_band_score, sel_band_score;
    int to_5ghz, to_6ghz;
    int cur_level, sel_level;
    unsigned int cur_est, sel_est;
    struct wpa_signal_info si;
    int cur_snr = 0;
    int ret = 0;

    // ... signal and throughput evaluation

    if (sel_est > cur_est + 5000) {
        wpa_dbg(wpa_s, MSG_DEBUG,
            "Allow reassociation - selected BSS has better estimated throughput");
        return 1;
    }

    to_5ghz = selected->freq > 4000 && current_bss->freq < 4000;
    to_6ghz = is_6ghz_freq(selected->freq) && !is_6ghz_freq(current_bss->freq);

    // ... band and signal difference evaluation

    cur_band_score = wpas_evaluate_band_score(current_bss->freq);
    sel_band_score = wpas_evaluate_band_score(selected->freq);
    min_diff += (cur_band_score - sel_band_score) * 2;

    if (wpa_s->signal_threshold && cur_level <= wpa_s->signal_threshold &&
        sel_level > wpa_s->signal_threshold)
        min_diff -= 2;

    diff = sel_level - cur_level;
    if (diff < min_diff) {
        wpa_dbg(wpa_s, MSG_DEBUG,
            "Skip roam - too small difference in signal level (%d < %d)",
            diff, min_diff);
        ret = 0;
    } else {
        wpa_dbg(wpa_s, MSG_DEBUG,
            "Allow reassociation due to difference in signal level (%d >= %d)",
            diff, min_diff);
        ret = 1;
    }

    return ret;
}
```

### Roaming Decision Criteria (wpa_supplicant)

Wpa_supplicant decides to roam based on several factors:

1. **Throughput Estimate**: If the selected BSS has an estimated throughput that is 5 Mbps higher than the current BSS, roam immediately.

   **Throughput Estimation Algorithm**:
   The `wpas_get_est_tpt()` function estimates throughput by:
   - **Rate Limiting Based on SNR**: Applies SNR-dependent rate limits to the maximum rate from the BSS
   - **HT (802.11n) Support**: If HT capabilities are detected, calculates HT20 and HT40 rates
   - **VHT (802.11ac) Support**: If VHT capabilities are detected, calculates VHT80 and VHT160 rates
   - **HE (802.11ax) Support**: If HE capabilities are detected, calculates HE rates
   - **EHT (802.11be) Support**: If EHT capabilities are detected, calculates EHT rates
   - **Channel Width Adjustments**: Adjusts SNR and rates based on channel width (20/40/80/160 MHz)
   - **Interpolation**: Uses min SNR-bitrate tables to interpolate rates between known SNR levels

   The estimation process uses pre-defined tables like `vht20_table`, `vht40_table`, `vht80_table`, `vht160_table`, `he80_table`, `he160_table`, and `eht80_table` that define minimum SNR requirements for each MCS (Modulation and Coding Scheme) rate.

2. **Band Preference**: Uses a band scoring system:
   - 6 GHz: score = 2 (highest)
   - 5 GHz: score = 1 (middle)
   - 2.4 GHz: score = 0 (lowest)

3. **Signal Strength Difference**:
   - Calculates a minimum required signal difference (`min_diff`) based on band scores
   - If the selected BSS has a signal strength difference greater than or equal to `min_diff`, roam
   - Adjusts `min_diff` based on throughput ratios (e.g., 1.5x, 1.2x, 1.1x, or just higher)

4. **Signal Threshold**: If current signal is below the configured threshold (-70 or -65 dBm) and the selected BSS has a signal above this threshold, reduce the required difference by 2 dB.

### Band Scoring Function

```c
// src/hostap/wpa_supplicant/events.c
static int wpas_evaluate_band_score(int frequency)
{
    if (is_6ghz_freq(frequency))
        return 2;
    if (IS_5GHZ(frequency))
        return 1;
    return 0;
}
```

## 2.1 802.11k / 802.11v: How they feed roaming (what does what)

This is the part that often gets misunderstood:

- **NetworkManager does not “implement” 802.11k or 802.11v roaming logic.** It mainly:
  - configures wpa_supplicant (e.g., `bgscan`)
  - watches `CurrentBSS` to notice that roaming happened
  - refreshes DHCP/IP config after a roam
- **wpa_supplicant (and sometimes the Wi-Fi driver) implement the 802.11k/v behavior**, and it can influence which BSS gets selected and how quickly scanning finds it.

### 802.11v (WNM BSS Transition Management / “BTM”)

When an AP wants to “nudge” a station to a better BSSID (or warn that disassociation is imminent), it can send a **BSS Transition Management Request**. In wpa_supplicant this is handled in:

- `/home/rebroad/src/hostap/wpa_supplicant/wnm_sta.c`
  - `ieee802_11_rx_bss_trans_mgmt_req()` (parses the request and candidate list)
  - `wnm_scan_process()` (applies candidate preferences/exclusions and picks a target BSS)
  - `wnm_set_scan_freqs()` (reduces scan to candidate frequencies when possible)
  - `wnm_is_bss_excluded()` (filters candidates based on AP-provided preferences)

**Why it can matter for decision making:** 11v can carry “preferred candidates” (and urgency like disassoc-imminent). That can materially change *which* BSSID is considered “best” and can trigger a focused scan rather than a broad scan.

**How it is exposed externally (observability):**

- wpa_supplicant defines ctrl_iface event prefixes such as `ESS_DISASSOC_IMMINENT`, `BSS_TM_QUERY`, and `BSS_TM_RESP` (see `/home/rebroad/src/hostap/src/common/wpa_ctrl.h`).
- wpa_supplicant updates a D-Bus property `BSS_TM_STATUS` (see `/home/rebroad/src/hostap/wpa_supplicant/dbus/dbus_new.c` and `/home/rebroad/src/hostap/wpa_supplicant/dbus/dbus_new_handlers.c`).

### 802.11k (RRM / Neighbor Reports)

802.11k is a family of “radio resource measurement” features; for roaming, the big practical piece is often **Neighbor Reports** (a structured list of nearby BSSIDs/channels).

In wpa_supplicant:

- `/home/rebroad/src/hostap/wpa_supplicant/rrm.c` implements RRM request/response handling, including Neighbor Report Requests and processing received reports.
- Neighbor report information can also arrive embedded in a 11v BTM request, and then gets used by `wnm_scan_process()` and `wnm_set_scan_freqs()` in `wnm_sta.c`.

**Why it can matter for decision making:** Neighbor reports can reduce scan time by focusing scans on the channels that are actually relevant, and can provide richer “candidate list” context than raw scan results alone.

**How it is exposed externally (observability):**

- wpa_supplicant defines ctrl_iface event prefixes `RRM_EVENT_NEIGHBOR_REP_RXED` and `RRM_EVENT_NEIGHBOR_REP_FAILED` (see `/home/rebroad/src/hostap/src/common/wpa_ctrl.h`).

## 2.2 What NetworkManager actually processes (and what it doesn’t)

NetworkManager currently does **not** appear to consume 802.11k neighbor reports or 802.11v BTM events for roaming decision-making. Instead, it:

- configures wpa_supplicant scanning behavior (bgscan; see `nm_supplicant_config_add_bgscan()` above)
- detects “roam happened” by watching `CurrentBSS` on the wpa_supplicant D-Bus interface:
  - `/home/rebroad/src/NetworkManager/src/core/devices/wifi/nm-device-wifi.c`
    - `supplicant_iface_notify_current_bss()` logs BSSID changes and renews DHCP when appropriate
- handles a limited set of wpa_supplicant D-Bus signals:
  - `/home/rebroad/src/NetworkManager/src/core/supplicant/nm-supplicant-interface.c`
    - `_signal_handle()` handles `BSSAdded`, `BSSRemoved`, `EAP`, `PskMismatch`, `SaePasswordMismatch`

If 802.11k/v are being used, the roam selection effect is happening inside wpa_supplicant/driver; NM is mostly “policy + lifecycle”.

## 3. Summary of Findings

### Initial BSSID Selection

NetworkManager selects the first compatible AP it finds in the scan results list. The list is unordered, with APs added to the end as they are discovered. This means the initial BSSID selection is essentially random based on discovery order.

### Roaming Decision

Roaming is primarily controlled by wpa_supplicant with NetworkManager configuring the bgscan parameters:

1. **When to Scan**:
   - For PSK: Scan every 30 seconds if signal < -70 dBm
   - For Enterprise: Scan every 30 seconds if signal < -65 dBm

2. **When to Roam**:
   - If new BSS has 5 Mbps higher estimated throughput
   - If new BSS has sufficient signal strength difference, considering band preference
   - 6 GHz > 5 GHz > 2.4 GHz in terms of band preference
   - Signal strength difference requirements are adjusted based on band and throughput ratios

### Band Preference

NetworkManager does favor 5 GHz and 6 GHz bands over 2.4 GHz, but this is implemented in wpa_supplicant's band scoring system rather than in NetworkManager itself.

## 4. Files Examined

- `/home/rebroad/src/NetworkManager/src/core/devices/wifi/nm-device-wifi.c`
- `/home/rebroad/src/NetworkManager/src/core/devices/wifi/nm-wifi-ap.c`
- `/home/rebroad/src/NetworkManager/src/core/supplicant/nm-supplicant-config.c`
- `/home/rebroad/src/NetworkManager/src/core/supplicant/nm-supplicant-interface.c`
- `/home/rebroad/src/hostap/wpa_supplicant/bgscan_simple.c`
- `/home/rebroad/src/hostap/wpa_supplicant/events.c`
- `/home/rebroad/src/hostap/wpa_supplicant/wnm_sta.c`
- `/home/rebroad/src/hostap/wpa_supplicant/rrm.c`
- `/home/rebroad/src/hostap/src/common/wpa_ctrl.h`
