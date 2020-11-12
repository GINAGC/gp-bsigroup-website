# Reverse Squid proxy rebuild

## Introdution 

The current reverse Squid proxy send everything to ICAP server
ICAP server take more time and resources to process unsupported files
Re-build Squid proxy to only send supported files to ICAP server
 
## Purpose

Only send supported files to ICAP
Save ICAP ' cost and resources
Redute ICAP processing time and reduce loading time

## Configuration

Define ACL list which allow forward to ICAP server, example below define ACL which only allow pdf file send to ICAP
```
acl pdffilter rep_header Content-Type ^[a-zA-Z]{1,}/pdf$
```
(condition pdffilter will be true, if the header Content-Type in the server response contains text like application/pdf).

Only allow request and response message match ACL pdffilter send to ICAP
```
adaptation_access gw_resp allow pdffilter
adaptation_access gw_resp deny all
adaptation_access gw_req allow pdffile
adaptation_access gw_req deny all
```
We can define list of ACL to match for the using purpose

Re-build Squid proxy to apply new configuration


