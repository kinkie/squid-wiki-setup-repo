# How to cache Vimeo

by Yuri Voinov

Warning: Any example presented here is provided "as-is" with no support
or guarantee of suitability. If you have any further questions about
these examples please email the squid-users mailing list.

## Outline

Sometimes you require to block (only) Vimeo videostreams.

## Usage

This blocks just Vimeo streams, not vimeo.com itself.

## Squid Configuration 

in your squid.conf
    acl vimeo url_regex pdl\.vimeocdn\.com\/.*\.mp4\? (akamai[hd|zed]|vimeocdn)\.[a-z]{3}\/.*\/(vimeo[.*\.mp4|.*\.m4s)
    http_access deny vimeo
