# Securing Kong API Gateway with CloudGuard WAF on Ubuntu LTS

## Introduction

This scenario demonstrates deployment of Kong API Gateway on Ubuntu LTS 22.04 system
and how to add Check Point CloudGuard WAF to protect the API Gateway.


## Initial enviroment

- Ubuntu LTS 22.04
- Kong API Gateway
- Check Point CloudGuard WAF agent Linux Embedded Profile installed later to protect it

We will use Killercoda scenario published [here](https://killercoda.com/94180bda-4838-456f-89db-fb7fef3a6b02/scenario/kong-appsec) as a base
that is basically fresh Ubuntu 22.04 LTS with Docker enabled to get started.
Follow scenario to bring demo application using Docker container:

```bash
# start new API exposed on 8080 - it was made to demonstrate bypass using SQL Injection on poorely built SQL query from unverified inputs
docker run -d -p 8080:5000  ghcr.io/openappsec/vulnerable-api-server:latest
# once ready, exposed on port 8080
docker ps
```

This API is exposing authenticated endpoint `/getemployees` 
with input field `email`
that is vulnerable to SQL Injection.

```bash
# expected behaviour returning records filtered by e-mail
curl -s -G -v -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJlbWFpbCI6ImpvaG4uc21pdGhAZXhhbXBsZS5jb20iLCJpZCI6MX0.pnlUuw6CzSG2In05n7WMDFP1l5GeqyAnWN98x9zcAc0" \
--data-urlencode "email=john.smith@example.com" \
http://localhost:8080/getemployees | jq .

# but we may get unfair advantage of SQL Injection strategy to build always true condition 
# that will return all records from the database table
curl -s -G -v -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJlbWFpbCI6ImpvaG4uc21pdGhAZXhhbXBsZS5jb20iLCJpZCI6MX0.pnlUuw6CzSG2In05n7WMDFP1l5GeqyAnWN98x9zcAc0" \
--data-urlencode "email=john.smith@example.com' OR '1'='1" \
http://localhost:8080/getemployees | jq .

# we basically finished email input and added additional SQL code that will always return true
```

## Kong API Gateway

Lets put API behind Kong API Gateway to have better control over it.
We will expose API on port 8000 and port 8080 should not be accessible directly
in production type of setup, because everything should be controlled by API Gateway.


```bash
# download and install Kong API Gateway
curl -Lo kong-3.3.1.amd64.deb "https://packages.konghq.com/public/gateway-33/deb/ubuntu/pool/focal/main/k/ko/kong_3.3.1/kong_3.3.1_amd64.deb" && sudo dpkg -i kong-3.3.1.amd64.deb

# configure Kong API Gateway declarative configuration in /etc/kong/kong.yml instead of database
sed -e '1idatabase = off' -e '1ideclarative_config = /etc/kong/kong.yml' /etc/kong/kong.conf.default > /etc/kong/kong.conf

# create declarative configuration file
# map vulnerable API to Kong API Gateway's root on port 8000
cat <<EOF > /etc/kong/kong.yml
_format_version: "2.1"
services:
- name: vulnerable-api-server
  url: http://localhost:8080
routes:
- name: vulnerable-api-server
  service: vulnerable-api-server
  paths:
  - /
EOF

# restart Kong API Gateway with new configuration
kong restart -c /etc/kong/kong.conf
```

Now we can access API through Kong API Gateway on port 8000:

```bash
# while Kong API Gateway is running on port 8000, we can access API through it
# still it is useful control point for many aspects like authentication, rate limiting, etc.
# but it does not protect from SQL Injection
curl -s -G -v -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJlbWFpbCI6ImpvaG4uc21pdGhAZXhhbXBsZS5jb20iLCJpZCI6MX0.pnlUuw6CzSG2In05n7WMDFP1l5GeqyAnWN98x9zcAc0" \
--data-urlencode "email=john.smith@example.com' OR '1'='1" \
http://localhost:8000/getemployees | jq .
```

## Check Point CloudGuard WAF

We will install Check Point CloudGuard WAF agent Linux Embedded Profile to protect Kong API Gateway.

Agents are installed based on agent token from Check Point [Infinity Portal](https://portal.checkpoint.com/).
Register to [portal](https://portal.checkpoint.com/register/) or open existing [CloudGuard WAF management app](https://portal.checkpoint.com/dashboard/appsec/cloudguardwaf#/waf-policy/getting-started).


Under Profiles create new  - choosing Linux Embedded Agent Profile.
![alt text](new-profiles.png)

Name profile `kong` and select Deployment `Kong application security`.
![alt text](kong-profile.png)

Make sure that you finish Publish and Enforce before continuing with agent installation.
Copy installation command from the profile. It loomks like this:

```bash
wget https://sc1.checkpoint.com/nanoagent/nanoegg -O nanoegg && chmod +x nanoegg && ./nanoegg --install --token cp-29f70c3e-bring-your-real-token-here
```

Once policy is enforced, return to API Gateway and run instakkation command.

Installed agent may be monitored in portal [Agents section](https://portal.checkpoint.com/dashboard/appsec/cloudguardwaf#/waf-policy/agents/) and it should be visible in the list.
Command `cpnano -s` on API gateway confirms WAF agent status and policy version enforced.

However we are still missing Asset for our API entry point to be enforced by WAF agent. Lets create new Web API Asset in the portal.

Follow asset creation wizard to create new asset for our API Gateway entry point.

We are mapping entry point (1) http://localhost:8000/ to the real API in backend (2) running on http://localhost:8080/. 
Note that mapping with Linux Embedded Profile is controlled by Kong configuration file `/etc/kong/kong.yml` and not by WAF agent configuration, so backedn URL in field (2) is optional.
![alt text](wizard1.png)

We are enabling ML based WAF protection, IPS engine and API discovery engines on next step. We would keep Learn/Detect in production for initial period of time, but we choose Prevent for demo.
![alt text](wizard2.png)

Make sure you choose existing `kong` profile for the asset. This is where asset is enforced.
![alt text](wizard3.png)

Finish rest of the wizard with default settings and confirm asset creation. Make sure to publish and ENFORCE the policy.

### Testing enforcement

Now we can test API Gateway again and see that WAF agent is protecting it from SQL Injection.

```bash
# agents section in portal confirms new policy? agent side:
cpnano -s

# you may wait for policy fetch in the loop
while true; do curl -s -G -v -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJlbWFpbCI6ImpvaG4uc21pdGhAZXhhbXBsZS5jb20iLCJpZCI6MX0.pnlUuw6CzSG2In05n7WMDFP1l5GeqyAnWN98x9zcAc0" --data-urlencode "email=john.smith@example.com' OR '1'='1" http://localhost:8000/getemployees ; sleep 5; done

# once policy is fetched and enforced, you should see WAF blocking SQL Injection attempts

# Monutor section / Important events in portal should show blocked SQL Injection attempts
# console should show WAF error message instead of JSON data
```

### Additinal API mapping in Kong

Kong API Gateway configuration is based on `services` and `routes` exposing APIs and services in declarative configuration file `/etc/kong/kong.yml`.

So our `kong.yml` could be extended with additional API mapping to the new backend service:

```bash
cat <<EOF > /etc/kong/kong.yml
_format_version: "2.1"
services:
- name: vulnerable-api-server
  url: http://localhost:8080
- name: ip-iol-cz-service
  url: http://ip.iol.cz/ip/
routes:
- name: vulnerable-api-server
  service: vulnerable-api-server
  paths:
  - /
- name: ip
  service: ip-iol-cz-service
  paths:
  - /ip/
EOF

# remember to check config and reload
kong check 
kong reload 
```

This is exposing service http://ip.iol.cz/ip/ through Kong API Gateway on path `/ip/`. So also http://localhost:8000/ip/ should be accessible and return source IP address.

```bash
# will return source IP address or your lab
curl -s -G -v http://localhost:8000/ip/ 

# BTW we may also simulate some WAF incident
curl -s -G -v http://localhost:8000/ip/?z=cat+/etc/passwd
```

## Conclusion

We have demonstrated how to secure API Gateway with Check Point CloudGuard WAF agent Linux Embedded Profile.
We have turned empty Linux into API Gateway with Kong and protected it with WAF agent.