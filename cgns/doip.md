
# DoIP - make meaning of dynamic_objects

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
FEEDNAME=my_feed2
dynamic_objects -n $FEEDNAME
# check it
dynamic_objects -l
# check it better
dynamic_objects -l | ./doip
# add an IP address to it
dynamic_objects -o $FEEDNAME -r 192.168.1.1 192.168.1.1 -a
# check it
dynamic_objects -l | ./doip

# add even more IPs
dynamic_objects -o $FEEDNAME -r 10.0.0.0 10.0.0.10 -a
dynamic_objects -l | ./doip
```

### Comparison

```bash
./doip <(dynamic_objects -l ) <(sleep 2; dynamic_objects -o feed2 -r 172.16.0.4 172.16.0.9 -d; dynamic_objects -l)
./doip <(dynamic_objects -l ) <(sleep 2; dynamic_objects -o feed2 -r 172.16.0.40 172.16.0.44 -a; dynamic_objects -l)
./doip <(dynamic_objects -l ) <(sleep 2; dynamic_objects -o feed2 -r 172.16.0.40 172.16.0.41 -d; dynamic_objects -l)

./doip <(dynamic_objects -l ) <(sleep 2; dynamic_objects -o feed2 -r 172.16.0.44 172.16.0.44 -d; dynamic_objects -l)

./doip <(dynamic_objects -efo_show ) <(sleep 10; dynamic_objects -efo_show)

./doip <(dynamic_objects -efo_update feed_demo; dynamic_objects -efo_show ) <(sleep 1; curl_cli -s -k -X PUT https://feed-serv.deno.dev/ip/192.168.1.98 ; dynamic_objects -efo_update feed_demo;dynamic_objects -efo_show)


./doip <(dynamic_objects -efo_update feed_demo; dynamic_objects -efo_show ) <(sleep 1; curl_cli -s -k -X DELETE https://feed-serv.deno.dev/ip/192.168.1.98 ; dynamic_objects -efo_update feed_demo;dynamic_objects -efo_show)

while true; do ./doip <(dynamic_objects -efo_update feed_demo; dynamic_objects -efo_show ) <(sleep 5;  dynamic_objects -efo_update feed_demo;dynamic_objects -efo_show); done


curl_cli -s -k -X PUT https://feed-serv.deno.dev/ip/192.168.1.98 
curl_cli -s -k -X DELETE https://feed-serv.deno.dev/ip/192.168.1.98 
curl_cli -s -k -X GET https://feed-serv.deno.dev/ip | jq . | grep 192.168.1.98
```