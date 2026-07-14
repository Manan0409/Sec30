# Day 13 - Writing Advanced Suricata Rules

## Objective

After learning how Suricata generates events and alerts, today I wanted to write more practical detection rules.

Instead of detecting every HTTP or DNS request, I focused on matching specific values inside the traffic like HTTP methods, URLs, User-Agent strings, and DNS queries.

---

## Understanding Rule Matching

Suricata first analyzes network traffic and creates protocol events.

For example:

- HTTP request
- DNS query
- TLS handshake

These events are stored inside `eve.json`.

After that, every enabled rule is checked against those events.

If all rule conditions match, Suricata generates an alert.

---

## Rule 1 - Detect curl User-Agent

The first rule detects requests made using curl.

```suricata
alert http any any -> any any (
msg:"SEC30 - curl detected";
flow:established,to_server;
http.user_agent;
content:"curl";
nocase;
sid:1000010;
rev:1;
)
```

### Testing

From Kali:

```bash
curl http://10.10.10.100
```

The request generated an HTTP event and my custom rule matched the User-Agent.


---

## Rule 2 - Detect Access to /admin

Next I created a rule to detect requests to the `/admin` page.

```suricata
alert http any any -> any any (
msg:"SEC30 - Admin page requested";
flow:established,to_server;
http.uri;
content:"/admin";
sid:1000011;
rev:1;
)
```

### Testing

```bash
curl http://10.10.10.100/admin
```

Whenever the requested URL contained `/admin`, Suricata generated an alert.


---

## Rule 3 - Detect HTTP POST Requests

Many login forms use the POST method to send credentials.

I created a rule to detect every POST request.

```suricata
alert http any any -> any any (
msg:"SEC30 - HTTP POST Request";
flow:established,to_server;
http.method;
content:"POST";
sid:1000012;
rev:1;
)
```

### Testing

```bash
curl -X POST http://10.10.10.100/login
```

The HTTP method matched `POST`, so Suricata generated an alert.


---

## Rule 4 - Detect GitHub DNS Lookup

Finally I created a DNS rule.

```suricata
alert dns any any -> any any (
msg:"SEC30 - GitHub DNS Lookup";
flow:to_server;
dns.query;
content:"github.com";
nocase;
sid:1000013;
rev:1;
)
```

### Testing

```bash
dig github.com
```

The DNS query matched the rule and generated an alert.


---

## Comparing eve.json and alerts.log

For every test I checked both log files.

### eve.json

Contains protocol events.

It shows information like:

- Timestamp
- Source IP
- Destination IP
- Protocol
- HTTP Method
- DNS Query
- User-Agent
- URL

Example:

```json
"event_type":"http"
"http_method":"GET"
"url":"/admin"
"http_user_agent":"curl/8.19.0"
```


---

### alerts.log

This file only contains alerts.

If a rule matches the event generated in `eve.json`, an alert appears here.

Example:

```
SEC30 - Admin page requested
```



---

## What I Learned

Today I learned that Suricata does not inspect traffic randomly.

Every rule tells Suricata:

- Which protocol to inspect
- Which field to inspect
- What value to search for

Only when all conditions match does it generate an alert.

I also understood the purpose of different keywords like:

- `flow`
- `content`
- `nocase`
- `http.uri`
- `http.method`
- `http.user_agent`
- `dns.query`

These keywords allow us to create very specific detection rules instead of alerting on every packet.

---

## Problems Faced

Some rules did not generate alerts initially because:

- The traffic did not match the rule conditions.
- The requested URL or HTTP method was different.
- The custom rules needed to be reloaded after making changes.

Testing each rule individually made it easier to find the issue.

---

## Final Result

Successfully created and tested multiple custom Suricata rules.

Verified detection for:

- HTTP User-Agent
- HTTP URI
- HTTP POST method
- DNS queries

This session helped me understand how detection engineers write signatures instead of only relying on community rule sets.