
# DoIP - make meaning of dynamic_objects

## Why

* `dynamic_objects` work with ranges - one cannot grep range for specific IP address
* it is difficult to figure out which IPs were added or removed recently to see changes easy way

## Solution

```bash
# without tool
dynamic_objects -l                           

    object name : LocalGatewayExternal
    range 0 : 172.16.0.4             172.16.0.4

    object name : LocalGatewayInternal
    range 0 : 172.16.1.5             172.16.1.5

    object name : feed1
    range 0 : 10.0.0.0               10.0.0.3
    range 1 : 192.168.1.1            192.168.1.4
    range 2 : 192.168.1.11           192.168.1.11

# tool expands ranges and brings feed per line
dynamic_objects -l | ./doip
# so grep is easy
dynamic_objects -l | ./doip | grep 172.16.0.4

    LocalGatewayExternal:172.16.0.4
    feedC:172.16.0.0,172.16.0.1,172.16.0.2,172.16.0.3,172.16.0.4,172.16.0.5

# it also makes comparison easy - presenting IPs or feeds added/removed between two states
./doip /tmp/do1.txt /tmp/do2.txt

    0:LocalGatewayInternal:
    0:feed1:
    0:feed2:
    0:my_feed:
    0:my_feed2:
    +:my_feed_demo:10.0.1.0,10.0.1.1,10.0.1.2
    0:LocalGatewayExternal:
```

## Install

```bash
# download and make executable
curl_cli -k -OL https://github.com/mkol5222/pub.filex/raw/refs/heads/cpfeedman/doip; chmod +x ./doip; 
# test
dynamic_objects -l | ./doip
```

## Simple scenario

```bash
# add new feed
FEEDNAME=my_feed_demo
dynamic_objects -n $FEEDNAME
# check it
dynamic_objects -l
# check it better
dynamic_objects -l | ./doip
dynamic_objects -l | ./doip | grep $FEEDNAME
# add an IP address to it
dynamic_objects -o $FEEDNAME -r 192.168.1.1 192.168.1.1 -a
# check it
dynamic_objects -l | ./doip
dynamic_objects -l | ./doip | grep $FEEDNAME

# add even more IPs
dynamic_objects -o $FEEDNAME -r 10.0.0.0 10.0.0.10 -a
dynamic_objects -l | ./doip
dynamic_objects -l | ./doip | grep $FEEDNAME
```

### Comparison

```bash
# capture current state
dynamic_objects -l | tee /tmp/do1.txt

# make changes
FEEDNAME=my_feed_demo
dynamic_objects -o $FEEDNAME -r 10.0.1.0 10.0.1.2 -a
# capture final state
dynamic_objects -l | tee /tmp/do2.txt

# compare states
./doip /tmp/do1.txt /tmp/do2.txt
./doip /tmp/do1.txt /tmp/do2.txt | grep :$FEEDNAME:

# one can use Unix trick with delay on second state to avoid temporary files
# new feed
dynamic_objects -n feedC
# populate it with IPs - notice second "file" has delay, change, and dump of final state
./doip <(dynamic_objects -l ) <(sleep 2; dynamic_objects -o feedC -r 172.16.0.0 172.16.0.10 -a; dynamic_objects -l)
# remove some IPs
./doip <(dynamic_objects -l ) <(sleep 2; dynamic_objects -o feedC -r 172.16.0.6 172.16.0.10 -d; dynamic_objects -l)
# add different range
./doip <(dynamic_objects -l ) <(sleep 2; dynamic_objects -o feedC -r 172.16.0.40 172.16.0.41 -a; dynamic_objects -l)
# current state
dynamic_objects -l | ./doip
# same with Unix trick
./doip <(dynamic_objects -l)
```

### Network feeds

Setup assumes following Network Feed defined and used in Security Gateway policy:

| Parameter | Value |
|-----------|-------|
| Name | `feed_demo` |
| URL | `https://feed-serv.deno.dev/ip` |
| Format | JSON |
| Data Type | IP Address |
| JSON query | `.[].ip` |



```bash
# it works with Network Feeds (efo - external feed objects)

# current feeds
./doip <(dynamic_objects -efo_show ) 
# were they updated in 10 seconds interval?
./doip <(dynamic_objects -efo_show ) <(sleep 10; dynamic_objects -efo_show)

# imagine you can influence feed contents with API
# list
curl_cli -s -k -X GET https://feed-serv.deno.dev/ip | jq -r '.[].ip'
# add and list
curl_cli -s -k -X PUT https://feed-serv.deno.dev/ip/192.168.1.98 
curl_cli -s -k -X GET https://feed-serv.deno.dev/ip | jq -r '.[].ip' | grep 192.168.1.98
# delete and list
curl_cli -s -k -X DELETE https://feed-serv.deno.dev/ip/192.168.1.98 
curl_cli -s -k -X GET https://feed-serv.deno.dev/ip | jq -r '.[].ip' | grep 192.168.1.98

# compare now and before for adding
./doip <(dynamic_objects -efo_update feed_demo; dynamic_objects -efo_show ) <(sleep 1; curl_cli -s -k -X PUT https://feed-serv.deno.dev/ip/192.168.1.98 ; dynamic_objects -efo_update feed_demo;dynamic_objects -efo_show)

# compare for deleting
./doip <(dynamic_objects -efo_update feed_demo; dynamic_objects -efo_show ) <(sleep 1; curl_cli -s -k -X DELETE https://feed-serv.deno.dev/ip/192.168.1.98 ; dynamic_objects -efo_update feed_demo;dynamic_objects -efo_show)

# you can also compare in one window, every five seconds
while true; do ./doip <(dynamic_objects -efo_update feed_demo; dynamic_objects -efo_show ) <(sleep 5;  dynamic_objects -efo_update feed_demo;dynamic_objects -efo_show); done
# and do chanegs of feed in other window manualy
curl_cli -s -k -X GET https://feed-serv.deno.dev/ip | jq -r '.[].ip'
curl_cli -s -k -X PUT https://feed-serv.deno.dev/ip/192.168.1.98 
curl_cli -s -k -X DELETE https://feed-serv.deno.dev/ip/192.168.1.98 
```