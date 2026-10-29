---
title: KI/PSE — lab IPsec tunel Cisco ISR 4321 ↔ MikroTik CCR1009
type: school
stav: aktualni
semestr: ZS 2026/27
tags: [school, pse, lab, ipsec, ikev2, cisco, mikrotik, routeros, netsketch]
created: 2026-10-29
updated: 2026-10-29
---

# Lab IPsec: Cisco ISR 4321 ↔ MikroTik CCR1009

← [[PSE|KI/PSE]]

> [!info] Deklarace využití AI
> Zadání (IPsec tunel mezi dvěma výrobci) dal vyučující. Topologii a volbu zařízení rozhodl Zdeněk. Připojení k MikroTiku a nastavení DHCP provedl Claude Sonnet 5.5 (Anthropic) přes SSH, ostatní konfigurace připravil Claude a **na zařízení zatím neběžely**. Vše pod nadpisem „Doplněno z Claude" je `Neověřeno` na hardwaru. Dle Směrnice rektora UJEP č. 7/2026 to pro výpisky není povinné (čl. 5.3/5.5), uvedeno dobrovolně.

## Zadání
Vytvořit IPsec tunel alespoň mezi dvěma technologiemi (dvěma výrobci). Zvoleno: **Cisco ISR 4321 (R1)** ↔ **MikroTik CCR1009-7G-1C-1S+**.

## Fyzické zapojení
```
PC1 ── R1 (ISR 4321) Gi0/0/0 ──▶ patch panel PP1 ──▶ MikroTik ether1   (WAN, tunel)
        Gi0/0/1 ── LAN A                              bridge ── LAN B ── PC2
```
- Patch panel PP1 (24× Cat6A, pasivní): v náčrtu NetSketch je port 5 (`70.089`) a 6 (`70.090`) pro R1 a CCR. Šablona je ve větvi `worktree-patch-panel` repa `zdenax/pse-app`.
- ⚠ V náčrtu byl jeden kabel z panelu do **Console** na R1. Pro tunel musí být do `Gi0/0/0`.

## Adresace (navrhovaná)

| Strana | Rozhraní | IP |
|---|---|---|
| WAN R1 | Gi0/0/0 | `10.0.0.1/30` |
| WAN MikroTik | ether1 (mimo bridge) | `10.0.0.2/30` |
| LAN A (R1) | Gi0/0/1 | `192.168.1.1/24` |
| LAN B (MikroTik) | bridgeLocal | `192.168.88.1/24` (DHCP `.10–.254`) |

Pre-shared key (jednorázový, jen pro lab, po labu reset zařízení): `ZMEN_MA_KLIC`

## Co je hotové (2026-10-29, přes SSH)

> [!success] MikroTik CCR1009 — RouterOS 7.6, hostname `MikroTik`
> - Připojeno přes **IPv6 link-local** (`ssh admin@fe80::ce2d:e0ff:fe0b:7e27%en8`), MAC připojení z WinBoxu na Macu selhalo (`MacConnection syn timeout`).
> - Konfigurace byla prázdná po resetu, všechny porty v `bridgeLocal`, firewall prázdný.
> - Záloha před změnou: `pre-dhcp.rsc` (na zařízení).
> - Nastaven **DHCP server** `dhcp1`. Mac dostal `192.168.88.254`.

```routeros
/export file=pre-dhcp
/ip address add address=192.168.88.1/24 interface=bridgeLocal
/ip pool add name=dhcp-pool ranges=192.168.88.10-192.168.88.254
/ip dhcp-server network add address=192.168.88.0/24 gateway=192.168.88.1 dns-server=192.168.88.1
/ip dhcp-server add name=dhcp1 interface=bridgeLocal address-pool=dhcp-pool disabled=no
```

> [!success] IPsec na MikroTiku nahrán (2026-10-29)
> `ether1` vyjmut z bridge, `10.0.0.2/30`, profil `ike2-prof`, peer `R1` (10.0.0.1), proposal `esp-prop`, identity s PSK a policy 192.168.88.0/24 ↔ 192.168.1.0/24. Záloha před změnou: `pre-ipsec.rsc`. **Tunel se ještě nenavázal, čeká se na Cisco** (R1 `Gi0/0/0` ↔ MikroTik `ether1` přes patch panel).

## Doplněno z Claude (Neověřeno na hardwaru)

### Cisco ISR 4321 (IOS XE, IKEv2 + crypto map): vložit do konzole R1

Blok je připravený ke kopírování celý najednou do `configure terminal` (vloží se po řádcích, `end` a `write memory` na konci).

> [!warning] Licence
> Potřebuje technologický balíček `securityk9`. Ověř `show version` / `show license`; případně `license boot level securityk9` a reload.

```
configure terminal
interface GigabitEthernet0/0/0
 ip address 10.0.0.1 255.255.255.252
 no shutdown
interface GigabitEthernet0/0/1
 ip address 192.168.1.1 255.255.255.0
 no shutdown
!
crypto ikev2 proposal PROP
 encryption aes-cbc-256
 integrity sha256
 group 14
crypto ikev2 policy POL
 proposal PROP
crypto ikev2 keyring KR
 peer MIKROTIK
  address 10.0.0.2
  pre-shared-key ZMEN_MA_KLIC
crypto ikev2 profile PROF
 match identity remote address 10.0.0.2 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR
!
crypto ipsec transform-set TS esp-aes 256 esp-sha256-hmac
 mode tunnel
!
ip access-list extended A-TO-B
 permit ip 192.168.1.0 0.0.0.255 192.168.88.0 0.0.0.255
!
crypto map CMAP 10 ipsec-isakmp
 set peer 10.0.0.2
 set transform-set TS
 set ikev2-profile PROF
 set pfs group14
 match address A-TO-B
!
interface GigabitEthernet0/0/0
 crypto map CMAP
end
write memory
```

### MikroTik CCR1009 (RouterOS 7.6): už nahráno, tady jako záznam

```routeros
# ether1 pryč z bridge a WAN adresa
/interface bridge port remove [find interface=ether1]
/ip address add address=10.0.0.2/30 interface=ether1
# IPsec (IKEv2) proti R1
/ip ipsec profile add name=ike2-prof hash-algorithm=sha256 enc-algorithm=aes-256 dh-group=modp2048 lifetime=1d
/ip ipsec peer add name=R1 address=10.0.0.1/32 exchange-mode=ike2 profile=ike2-prof
/ip ipsec proposal add name=esp-prop auth-algorithms=sha256 enc-algorithms=aes-256-cbc pfs-group=modp2048 lifetime=1h
/ip ipsec identity add peer=R1 auth-method=pre-shared-key secret="ZMEN_MA_KLIC"
/ip ipsec policy add src-address=192.168.88.0/24 dst-address=192.168.1.0/24 tunnel=yes peer=R1 proposal=esp-prop sa-src-address=10.0.0.2 sa-dst-address=10.0.0.1
```
Pokud by se někdy přidal `masquerade`, přidej před něj výjimku: `/ip firewall nat add chain=srcnat action=accept src-address=192.168.88.0/24 dst-address=192.168.1.0/24 place-before=0`.

> [!tip] Shoda parametrů
> Na obou stranách: IKEv2, AES-256, SHA-256, DH skupina 14 (`modp2048`), PSK, tunelový režim, shodné „zajímavé" podsítě 192.168.1.0/24 ↔ 192.168.88.0/24. Nejčastější chyba je prohozená ACL / policy nebo jiná skupina DH.

### Ověření

| Kde | Příkaz | Co čekat |
|---|---|---|
| Cisco | `show crypto ikev2 sa` | stav `READY` |
| Cisco | `show crypto ipsec sa` | `#pkts encaps/decaps` rostou po pingu |
| MikroTik | `/ip ipsec active-peers print` | peer `R1`, stav `established` |
| MikroTik | `/ip ipsec installed-sa print` | dvě SA (in/out) |
| PC1 → PC2 | `ping 192.168.88.x` | odpovídá; první paket může propadnout při navazování |

### Poznámky k přístupu
- Připojení k MikroTiku bez IP adresy: IPv6 link-local z EUI-64 MAC `fe80::<MAC s převráceným 7. bitem>%en8`. Adresa rozhraní (bridge) se liší od MAC, která se ukazuje ve WinBoxu (Neighbors).
- Po labu se zařízení vrací na default konfiguraci (reset). Heslo `admin` je slabé, po cvičení změnit.

## Související
- [[lab-trojuhelnik-2026-10-15]] — R1 Cisco ISR 4321 v předchozím labu
