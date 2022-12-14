---
categories: Feature
---
# Feature: Gzip compression / decompression in Squid

- **Goal**: To make Squid capable of compressing or decompressing
    objects in transit.
- **Status**: a partial 3rd-party solution exists, needs testing
- **Version**: 3.1
- **Developer**:
- **More**: [3rd party eCAP
    module](/ThirdPartyModules/EcapGzip)

## Details

An [eCAP](/Features/eCAP)
module developed by Constantin Rack at VIGOS to support gzip compression
of text/html cache *misses* is
[available](/ThirdPartyModules/EcapGzip).
That 3rd-party module has not been tested by the Squid Project.

