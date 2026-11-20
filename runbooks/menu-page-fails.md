# Menu page fails to load (restaurant list never shows)

## How you know

- Customers report the app "keeps loading" and the restaurant list never
  appears; some get an error page.
- From outside the server, the API hangs and then gives up:

  ```
  $ curl -v http://localhost:18080/api/restaurants
  * Request completely sent off
  curl: (28) Operation timed out after 8005 milliseconds with 0 bytes received
  ```
- On the server, `apt-get update` and any outbound call by hostname fail,
  because DNS lookups time out.

**Is it happening again? (under a minute)** Run these four commands. Any one
that doesn't match the healthy result points straight at the fault.

```bash
curl -s -m 5 -o /dev/null -w '%{http_code}\n' http://localhost:18080/api/restaurants  # healthy: 200
docker exec -u oncall bukka-prod-1 ss -ltn | grep 8080                       # healthy: 0.0.0.0:8080, NOT 127.0.0.1:8080
docker exec -u oncall bukka-prod-1 sudo iptables -S INPUT | grep 8080        # healthy: -A INPUT -p tcp ... --dport 8080 -j ACCEPT
docker exec -u oncall bukka-prod-1 getent hosts github.com                   # healthy: prints an IP address
```

## Check first

Work from the outside in. Log in with `docker exec -it -u oncall bukka-prod-1 bash`.

1. **Can customers reach it at all?** From your machine:
   `curl -v -m 8 http://localhost:18080/api/restaurants`.
   Healthy: a JSON list of restaurants. A **timeout** means something is
   silently dropping packets (usually a firewall). A **"Connection refused"**
   means the packet arrived and nothing is listening on that address and port.
2. **Is the app running?** `bukka-api status` should say `running`, and
   `bukka-api logs` should show `Uvicorn running on http://0.0.0.0:8080`.
   If the logs say `http://127.0.0.1:8080`, it only accepts connections
   from the server itself.
3. **Which address is it listening on?** `ss -ltnp`. Healthy:
   `0.0.0.0:8080` (all interfaces). `127.0.0.1:8080` means localhost only.
4. **Does it answer locally, and on the real network address?**
   `curl -s http://127.0.0.1:8080/api/restaurants` (local) and
   `curl -sv http://$(hostname -i):8080/api/restaurants` (eth0). If the first works but the
   second says **Connection refused**, it's the bind address.
5. **Is the firewall dropping traffic?** `sudo iptables -L INPUT -n -v --line-numbers`.
   Healthy: policy `DROP` **and** an `ACCEPT tcp dpt:8080` rule. Packet
   counters going up on the policy line while you curl means your requests
   are being dropped.
6. **Does DNS work?** `cat /etc/resolv.conf`, then `dig github.com`.
   `communications error to <ip>#53: timed out` means the configured
   resolver is unreachable. Compare with `dig @1.1.1.1 github.com +short`:
   if that answers, the network is fine and only the resolver setting is wrong.

## Fix

Fix all three; fixing one reveals the next. Don't turn the firewall off.

1. **Bind the API to all interfaces** (the setting is read on every start,
   so this survives `bukka-api restart`):
   ```bash
   sudo cp /etc/bukka/api.env /etc/bukka/api.env.bak
   sudo sed -i 's/^BUKKA_HOST=127.0.0.1$/BUKKA_HOST=0.0.0.0/' /etc/bukka/api.env
   ```
2. **Allow only the API port through the firewall**, keeping default-deny:
   ```bash
   sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
   sudo iptables -S INPUT   # -P INPUT DROP must still be the first line
   ```
3. **Point DNS back at a working resolver.** On Docker Desktop that's
   Docker's own resolver, `192.168.65.7` (check what a fresh container gets
   with `docker run --rm --entrypoint cat bukka-lab-buk21 /etc/resolv.conf`):
   ```bash
   sudo cp /etc/resolv.conf /etc/resolv.conf.bak
   echo "nameserver 192.168.65.7" | sudo tee /etc/resolv.conf
   ```
4. **Restart and confirm:** `bukka-api restart`, then run the four
   "Is it happening again?" commands above. Run `bukka-api restart` once
   more and check again, to prove the fixes stick.
5. On a real server, also persist the firewall rule so it survives a reboot:
   `sudo sh -c 'iptables-save > /etc/iptables/rules.v4'` (needs the
   `iptables-persistent` package).

## Why it happens

The overnight "security policy" change did three things. **(1)** It set
`BUKKA_HOST=127.0.0.1` in `/etc/bukka/api.env`, so the API only listens on
localhost. Anything arriving on the server's real address (`eth0`) gets a
TCP reset: **Connection refused**. A refusal is a fast, explicit "no one
is listening here", which proves the network path works and the problem is
the app's bind address. **(2)** It made the firewall default-deny
(`INPUT` policy `DROP`) but only allowed SSH (22) and ICMP, not the API
port 8080. A `DROP` rule discards packets without replying, so the client
waits until it gives up: **connection timed out**. A timeout means
"something in between ate the packet", which points at a firewall, not at
the app. That timeout is what customers saw first; the bind address fault
was hidden behind it until the port was opened. **(3)** It replaced
`/etc/resolv.conf` with `nameserver 192.0.2.53`, an address range reserved
for documentation that nobody answers. DNS queries time out, so `apt` and
outbound calls by hostname fail, even though the network itself works
(`dig @1.1.1.1` succeeds).
