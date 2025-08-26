
# Secure Downloads on GaiaOS

How to use `curl` (`curl_cli`) on GaiaOS in a secure way.

```bash


# On GaiaOS, curl is available as curl_cli
curl_cli -v

# Unfortunately, curl_cli does not have a trusted CA bundle by default
curl_cli https://ifconfig.me
# curl: (60) SSL certificate problem: unable to get local issuer certificate

# It may be tempting to use --insecure (also known as -k)
curl_cli --insecure https://ifconfig.me
curl_cli -k https://ifconfig.me

# But the CORRECT WAY is to provide a valid, Internet-trusted CA bundle, like:
curl_cli --cacert $FWDIR/database/ca_bundle.pem https://ifconfig.me

# There is also a bundle provided by curl at https://curl.se/ca/cacert.pem
#   derived from Mozilla’s trusted CA list (the same CAs trusted by Firefox)
curl_cli --cacert $FWDIR/database/ca_bundle.pem -o /tmp/cacert.pem https://curl.se/ca/cacert.pem

# Then use:
curl_cli --cacert /tmp/cacert.pem https://ifconfig.me

# You can create a convenient alias:
alias curl='curl_cli --cacert $FWDIR/database/ca_bundle.pem'
# Check:
which curl
# Use:
curl https://ifconfig.me

```