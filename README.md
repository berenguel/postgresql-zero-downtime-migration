# postgresql-zero-downtime-migration
This is a one stop shop for various strategies to copy data between PostgreSQL instances with near zero downtime.

Side-by-side approach is the safest and fastest solution for these complex database transitions:

Use Cases:
- Major Version Upgrades: Move from an older to a newer PostgreSQL version (e.g., PG13 to PG16) without a downtime-inducing dump/restore or lengthy in-place upgrade.

- Networking & Endpoint Changes: Seamlessly switch to a new VNET, subnet, IP range, or cloud region that would otherwise require service interruption.

- Hardware / Machine Generation: Migrate to a more powerful instance or faster storage (e.g., non-confidential compute to confidential compute) without impacting live traffic.

- Infrastructure Restructuring: Consolidate multiple databases onto a single large instance, or split a monolithic database into dedicated instances for better performance and isolation.
