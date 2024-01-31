---
layout: default
title: Update Guide
parent: Deploy with Starburst Admin
grand_parent: SEP Deployment Mechanisms
nav_order: 5
permalink: /docs/deployment-mechanisms/starburst-admin/update-guide/
---

# Update Starburst Admin

Like the Starburst Admin {doc}`installation process <installation-guide>`, the
Starburst Admin update process uses the `ansible-galaxy` tool.

:::{note}
The `ansible-galaxy` tool replaces all existing files in the
`~/.ansible/collections/ansible_collections/starburst/admin/` directory with the
latest files from the updated package.
:::

Use the following steps to update Starburst Admin:

1. Download the package for your desired version of Starburst Admin.
2. Back up all the configuration files you created or modified during
   the installation process.
3. Use the following command to install the new version of Starburst Admin:
    ```shell
    ansible-galaxy collection install starburst-admin-*.tar.gz
    ```
4. Restore your backed-up files to their original locations.
5. Restart SEP using the `stop.yml` and `start.yml` playbooks.

