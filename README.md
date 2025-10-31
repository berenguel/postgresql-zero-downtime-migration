# postgresql-zero-downtime-migration
This is a one stop shop for various strategies to copy data between PostgreSQL instances with near zero downtime.

A side-by-side approach is the fastest solution, allowing for minimal downtime, for these complex database transitions:

Use Cases:
- Major Version Upgrades (MVU): Move from an older to a newer PostgreSQL version (e.g., PG13 to PG16) without a downtime-inducing dump/restore or lengthy in-place upgrade.
  > [MVU](https://github.com/berenguel/postgresql-zero-downtime-migration/blob/main/postgresql-side-by-side-migration-mvu.md)
  
- Networking & Endpoint Changes: Seamlessly switch to a new VNET, subnet, IP range, or cloud region that would otherwise require service interruption.
- Hardware / Machine Generation: Migrate to a more powerful instance or faster storage (e.g., non-confidential compute to confidential compute) without impacting live traffic.
- Infrastructure Restructuring: Consolidate multiple databases onto a single large instance, or split a monolithic database into dedicated instances for better performance and isolation.
  >[General Migratiion](https://github.com/berenguel/postgresql-zero-downtime-migration/blob/main/postgresql-side-by-side-general-use-case.md)
  
- Blue Green Deployments: a software release strategy used primarily to minimize downtime and reduce risk during application updates, configuration changes, or underlying architectural migrations. 
  > [Blue/Green](https://github.com/berenguel/postgresql-zero-downtime-migration/blob/main/blue-green-architectural-changes.md)
