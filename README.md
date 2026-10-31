# SKRED Messenger — Hard-coded AES fallback key (fixed)

> **Full illustrated write-up:** https://hoseiny.de/en/cvd/skred/

**Status:** Fixed. Coordinated via the German **BSI CERT-Bund** (ticket CERT-Bund#2026030328001085) and **CERT/CC** (VU#629507). The vendor responded personally, engaged on the technical details, and shipped fixes.
**Severity:** Low–Medium — **local, device-bound**. The issue was a *fallback path*, required access to the device itself, and did not expose message contents. No remote attack.
**Vendor:** SKRED SAS / Twinlife — SKRED Messenger (Android / iOS, open source, AGPL)
**Reporter:** Abbas Hoseiny — independent · abbas@hoseiny.de

## TL;DR

SKRED's key-management layer (`KeyChain`) protects locally stored "secured" settings with a key from the Android Keystore. When Keystore key creation failed — a known problem on some older devices/ROMs — the app fell back to a **hard-coded key that was identical on every installation**, computed deterministically from two byte arrays in the (open) source. Anyone with **local access** to such a device could decrypt those local settings.

This is a constructive-outcome case: the vendor acknowledged the report and reworked the key management so the fallback key is **no longer used to encrypt**. This repository documents the finding and the fix; the full write-up — including the coordination story — is on hoseiny.de.

## The finding — F-012

On a device where the Android Keystore failed, the app derived a fallback AES key deterministically from two byte arrays baked into the source, XOR-combined. The result was a 128-bit key that was the **same on every installation** and reconstructible from the open source:

```java
// simplified from the open-source KeyChain implementation
static Key getDefaultSecretKey() {
    int n = Math.min(UUID1.length, UUID2.length);
    byte[] key = new byte[n];
    for (int i = 0; i < n; i++)
        key[i] = (byte) (UUID1[i] ^ UUID2[n - 1 - i]);
    return new SecretKeySpec(key, "AES");   // 128-bit, same on every device
}
```

## Impact — honestly scoped

- It was a **fallback path**, not the normal encryption route (which uses a per-device Keystore key).
- It affected **locally stored settings**, not the contents of everyone's messages.
- Exploiting it required **access to the device itself** — not a remote attack.

So the severity lives in the local, device-bound case. No inflated score is attached here.

## The fix

The vendor reworked `KeyChain`:

- The fallback key is **no longer used to encrypt**. If Keystore key creation fails, the app **fails closed** (error state) instead of encrypting with the known key.
- The key now only **decrypts legacy data during migration**.
- Support for Android 5.0/5.1 (the main source of the broken Keystore) was **dropped**.

Verified in the vendor's public sources.

## Also addressed — F-009 (certificate pinning)

The app previously shipped without a network security configuration. The vendor added one that trusts only system certificates, disables cleartext traffic, and pins a real pin-set for its own domain. Confirmed in the published app.

## Coordination & credit

Reported through the BSI CERT-Bund and tracked at CERT/CC (VU#629507); the vendor was coordinated through the CERT rather than contacted directly. The vendor's technical lead replied in person and in detail, shared additional architecture documentation, pointed to the open-source code base, and shipped the fixes promptly. This is coordinated disclosure working the way it should.

## Scope of this write-up

This covers only the items that are **fixed and verifiable** in the vendor's public sources. It is published to document a constructive coordinated-disclosure outcome — not to provide an exploit.

---

*Part of a small series of coordinated-disclosure write-ups: https://hoseiny.de/en/cvd/*
