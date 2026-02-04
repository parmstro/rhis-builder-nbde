# rhis-builder-nbde

Deprecated. Use rhis-builder-quadlet-deploy.
The functionality of rhis-builder-nbde has been normalized to use a basic quadlet deploy role. All functionality here should directly translate to the new rhis-builder-quadlet-deploy roles. This uses the same configuration parameters as rhis-builder-nbde.

NOTE: there are three new variables for the container definition to use with the quadlet_deploy role. Essentially the components from verify_deployment.yml
```
test_shell_command: "echo 'OcelotPantherJaguarBarnCat' | clevis encrypt tang '{\"url\":\"{{ ansible_fqdn }}:8080\"}' -y | clevis decrypt"
test_value: "OcelotPantherJaguarBarnCat"
test_expected_result: "OcelotPantherJaguarBarnCat"
```

***
**NOTE:** All ansible variables, files and templates for rhis-builder-* repos are now configured through the unified inventory project [rhis-builder-inventory](https://github.com/parmstro/rhis-builder-inventory)

The code in this repo builds out the tang and clevis deployment for RHIS using RHEL container hosts
