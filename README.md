# linux-dns-setup- script


আপনার অনুরোধ অনুযায়ী RHEL 8-এ DNS সার্ভার সেটআপের পুরো প্রক্রিয়াটি একটি Bash স্ক্রিপ্ট আকারে দেওয়া হলো। এই স্ক্রিপ্টটি `bind` ইনস্টল করা থেকে শুরু করে জোন ফাইল তৈরি এবং সার্ভার চালু করার সব ধাপ সম্পন্ন করবে।

```bash
#!/bin/bash

# স্ক্রিপ্টটি root ইউজার হিসেবে চালাতে হবে
if [ "$EUID" -ne 0 ]; then
  echo "Please run as root"
  exit
fi

# Step 1: Install bind and bind-utils
echo "Installing bind and bind-utils..."
dnf install -y bind bind-utils

# Step 2: Backup the original named.conf
echo "Backing up the named.conf file..."
cp /etc/named.conf /etc/named.conf.backup

# Step 3: Modify named.conf
echo "Configuring /etc/named.conf..."
cat <<EOL > /etc/named.conf
options {
    listen-on port 53 { 127.0.0.1; 192.168.1.10; };  # আপনার সার্ভারের IP
    directory     "/var/named";
    dump-file     "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    allow-query     { localhost; 192.168.1.0/24; };  # আপনার নেটওয়ার্ক IP
    recursion yes;

    dnssec-enable yes;
    dnssec-validation yes;
};

logging {
    channel default_debug {
        file "data/named.run";
        severity dynamic;
    };
};
EOL

# Step 4: Configure zone in /etc/named.rfc1912.zones
echo "Configuring /etc/named.rfc1912.zones..."
cat <<EOL >> /etc/named.rfc1912.zones
zone "example.com" IN {
    type master;
    file "/var/named/example.com.forward";
    allow-update { none; };
};

zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "/var/named/example.com.reverse";
    allow-update { none; };
};
EOL

# Step 5: Create forward and reverse zone files
echo "Creating forward and reverse zone files..."
cat <<EOL > /var/named/example.com.forward
\$TTL 86400
@   IN  SOA ns1.example.com. admin.example.com. (
        2023092001  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400 )     ; Minimum TTL
@   IN  NS  ns1.example.com.
@   IN  A   192.168.1.10
ns1 IN  A   192.168.1.10
EOL

cat <<EOL > /var/named/example.com.reverse
\$TTL 86400
@   IN  SOA ns1.example.com. admin.example.com. (
        2023092001  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400 )     ; Minimum TTL
@   IN  NS  ns1.example.com.
10  IN  PTR example.com.
EOL

# Step 6: Update file permissions
echo "Updating file permissions..."
chown root:named /var/named/example.com.forward /var/named/example.com.reverse
chmod 640 /var/named/example.com.forward /var/named/example.com.reverse

# Step 7: Update firewall rules
echo "Updating firewall rules..."
firewall-cmd --add-service=dns --permanent
firewall-cmd --reload

# Step 8: Start and enable named service
echo "Starting and enabling named service..."
systemctl start named
systemctl enable named

# Step 9: Check named service status
echo "Checking named service status..."
systemctl status named

echo "DNS server setup completed successfully!"
```

### কিভাবে চালাবেন:

1. একটি নতুন ফাইল তৈরি করুন, উদাহরণস্বরূপ: `setup_dns.sh`
   
   ```bash
   vi setup_dns.sh
   ```

2. এই স্ক্রিপ্টটি ফাইলে পেস্ট করুন।

3. ফাইলটিকে executable করতে নিচের কমান্ডটি দিন:

   ```bash
   chmod +x setup_dns.sh
   ```

4. স্ক্রিপ্টটি চালানোর জন্য:

   ```bash
   sudo ./setup_dns.sh
   ```

### এই স্ক্রিপ্টটি কি করবে:
- `bind` এবং `bind-utils` ইনস্টল করবে।
- `named.conf` কনফিগার করবে।
- ফরোয়ার্ড এবং রিভার্স জোন ফাইল তৈরি করবে।
- ফায়ারওয়াল কনফিগার করবে।
- `named` সার্ভিস চালু ও সক্রিয় করবে।

এই স্ক্রিপ্টটি সম্পূর্ণ প্রক্রিয়া স্বয়ংক্রিয় করবে, আপনাকে আলাদা করে কিছু কনফিগার করার প্রয়োজন নেই।
