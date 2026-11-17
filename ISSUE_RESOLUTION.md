# Issue Resolution: mac_address excludes the highest valid unicast (0xFE) and multicast (0xFF) first octets

## Metadata
- **Target Repository**: `joke2k/faker`
- **Private Repository**: `rajanjava/faker-private`
- **Upstream Issue**: [#2504: mac_address excludes the highest valid unicast and multicast first octets](https://github.com/joke2k/faker/issues/2504)
- **Status**: Resolved & Verified

---

## Problem Description & Root Cause
In `faker.providers.internet.Provider.mac_address(multicast=False)`:
```python
mac = [self.generator.random.randint(0x00, 0xFF) for _ in range(0, 5)]
if multicast is True:
    mac.insert(0, self.generator.random.randrange(0x01, 0xFF, 2))
else:
    mac.insert(0, self.generator.random.randrange(0x00, 0xFE, 2))
return ":".join("%02x" % x for x in mac)
```

In Python's `random.randrange(start, stop, step)`, the `stop` parameter is exclusive.
- For unicast MAC addresses, the least significant bit (LSB) of the first octet must be `0` (even numbers: `0x00, 0x02, ..., 0xFE`). There are 128 valid even values in an 8-bit octet (`0x00` through `0xFE` / 254). Because `stop=0xFE` (254) was used, `0xFE` was excluded from `range(0x00, 0xFE, 2)`, so the highest possible value generated was only `0xFC` (252).
- For multicast MAC addresses, the LSB of the first octet must be `1` (odd numbers: `0x01, 0x03, ..., 0xFF`). There are 128 valid odd values in an 8-bit octet (`0x01` through `0xFF` / 255). Because `stop=0xFF` (255) was used, `0xFF` was excluded from `range(0x01, 0xFF, 2)`, so the highest possible value generated was only `0xFD` (253).

As a result, valid boundary octets (`fe` for unicast and `ff` for multicast) were permanently excluded from generated MAC addresses.

---

## Solution
Change the exclusive upper bound `stop` to `0x100` (256) for both parity-preserving `randrange` calls:
1. `self.generator.random.randrange(0x00, 0x100, 2)`: produces all 128 even numbers from `0x00` through `0xFE` (254).
2. `self.generator.random.randrange(0x01, 0x100, 2)`: produces all 128 odd numbers from `0x01` through `0xFF` (255).

---

## Code Changes
```diff
diff --git a/faker/providers/internet/__init__.py b/faker/providers/internet/__init__.py
index 2090df9..e3c2b26 100644
--- a/faker/providers/internet/__init__.py
+++ b/faker/providers/internet/__init__.py
@@ -672,9 +672,9 @@ class Provider(BaseProvider):
         """
         mac = [self.generator.random.randint(0x00, 0xFF) for _ in range(0, 5)]
         if multicast is True:
-            mac.insert(0, self.generator.random.randrange(0x01, 0xFF, 2))
+            mac.insert(0, self.generator.random.randrange(0x01, 0x100, 2))
         else:
-            mac.insert(0, self.generator.random.randrange(0x00, 0xFE, 2))
+            mac.insert(0, self.generator.random.randrange(0x00, 0x100, 2))
         return ":".join("%02x" % x for x in mac)
```

---

## Verification
- Added `test_mac_address_highest_octets` in `tests/providers/test_internet.py` testing both deterministic argument bounds (`randrange(0x00, 0x100, 2)` and `randrange(0x01, 0x100, 2)`) and empirical generation of the highest valid first octets (`fe` and `ff`).
- Verified failure prior to fix:
  `AssertionError: mac_address() unicast failed to generate highest valid first octet 'fe'`
- Verified pass after fix:
  `100 passed in 1.34s`
