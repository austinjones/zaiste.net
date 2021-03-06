
+++

+++
# How To Setup fail2ban Ubuntu 18.04

Fail2ban is an application that parses the system logs looking for signs of an automated attack.

When an attack attempted is identified, Fail2ban adds a rule to iptables to block the IP address of the attacker. This block can be set either for a specific period of time or permanently. fail2ban focuses on SSH attacks.

Since fail2ban 0.9, the bans are persistent. fail2ban maintains a database at `/var/lib/fail2ban/fail2ban.sqlite3`.

## fail2ban: Installation

```bash
sudo apt-get install fail2ban
```

## fail2ban: Configuration

The `/etc/fail2ban/fail2ban.conf` contains the default configuration. It is a
good practice to copy this file to `/etc/fail2ban/fail2ban.local`. In fail2ban
`.local` files override `.conf` files

```bash
cp /etc/fail2ban/fail2ban.conf /etc/fail2ban/fail2ban.local
```

The `/etc/fail2ban/jail.conf` file enables fail2ban configuration for SSH.
Again, it is a good practice to copy this file to `/etc/fail2ban/jail.local`.

## fail2ban: Whitelist IPs

To make an IP ignored by fail2ban, use the `ignoreip` setting:

```bash
ignoreip = 127.0.0.1/8 1.1.1.1
```

## fail2ban: Ban Time

Use `bantime` setting to set the number of seconds for which an IP is banned. If set to a negative number, the ban is permanent.

## fail2ban: Retry Time / Amount

Use `findtime` to set the number of seconds between login attempts before a ban is set. It is used in conjunction with `maxretry` which sets how many attempts can be made to access the server from a specific IP before it is banned.

## fail2ban: SSH Settings

By default in fail2ban only SSH settings are enabled.

```bash
[ssh]

enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 6
```

For `port`, if using the default `22` port, the service name can be used. Otherwise, for non-traditional ports, use the port number explicitly.

## fail2ban: Regex

fail2ban uses regular expressions (regex) to parse log files in order to identify potential attacks. fail2ban uses Python’s regex implementation.

Use `fail2ban-regex` to check if your custom filter is working.

```bash
fail2ban-regex /var/www/html/logs/access.log /etc/fail2ban/filter.d/wordpress.conf
```

## fail2ban: Command Line Client

fail2ban provides a command-line (CLI) tool `fail2ban-client`.

+ `start` - start the fail2ban server and all defined jails.
+ `reload` - reload the fail2ban configuration
+ `reload <jail>` - reload a specific jail configuration
+ `stop` - stop the fail2ban server
+ `status` - show the fail2ban status and active jails


