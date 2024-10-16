# DNS Master and slave

configuiration of DNS master and slave

# `/etc/default/named`

Disable IPv6 to avoid errors 

```
# startup options for the sever
OPTIONS="-u bind -4"
```