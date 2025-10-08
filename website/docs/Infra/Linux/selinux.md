# SE Linux

## Allowing nginx proxy_pass on local

Mainly handy for `proxy_pass` into docker containers

Problem can be identified by inspecting nginx error logs or running `sudo cat /var/log/audit/audit.log | grep nginx | grep denied`.

```
# NGINX log
2025/02/06 18:22:02 [crit] 224940#224940: *1 connect() to 127.0.0.1:8200 failed (13: Permission denied) while connecting to upstream, client: 10.9.11.33, server: myapp.myhost.com, request: "GET / HTTP/1.0", upstream: "http://127.0.0.1:8200/", host: "myapp.myhost.com"

# Audit log
type=AVC msg=audit(1738866198.971:3658): avc:  denied  { name_connect } for  pid=225008 comm="nginx" dest=8200 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:trivnet1_port_t:s0 tclass=tcp_socket permissive=0
```

To remedy the problem, run `sudo setsebool -P httpd_can_network_connect 1`
