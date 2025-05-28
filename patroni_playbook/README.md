# Setup

1. Set your own hosts to `inventory` file
   NOTE: one host MUST have `keepalived_master=true` set
2. Set keepalived ip address from your subnet as Ansible variable `keepalived_ip`
3. *(optional)* Set other variables
# Variable list
```
patroni_postgres_root_password
patroni_postgres_replicator_password
keepalived_password
keepalived_ip # REQUIRED
```
# Default passwords
Postgres root - `rootpassword`

Postgres replicator - `reppassword`

Keepalived - `password`