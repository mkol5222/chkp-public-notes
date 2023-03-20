
## Updatable objects in Check Point rulebase

### Management API

Documented at [Management API Reference](https://sc1.checkpoint.com/documents/latest/APIs/#introduction~v1.9%20) - search for *updatable*

### Retrieve with API

Update contents of Updatable objects reporsitory from User Center
```bash
# ssh cp179
mgmt_cli -r true update-updatable-objects-repository-content
```

### Search updatable objects repository
```bash
# ssh cp179
mgmt_cli -r true show updatable-objects-repository-content filter.text "Amazon"  --format json
# note results are paged - from 1 to 50 of XYZ total

# max limit 500 and full detail
mgmt_cli -r true show updatable-objects-repository-content filter.text "Amazon"  limit 500 details-level full --format json 

# list from URI and name
#   name-in-updatable-objects-repository additional-properties.uri

# notice response structure
mgmt_cli -r true show updatable-objects-repository-content filter.text "Amazon"  limit 500 details-level full --format json | jq '. | keys'
# notice details of first objects
mgmt_cli -r true show updatable-objects-repository-content filter.text "Amazon"  limit 500 details-level full --format json | jq -r '.objects[0]'
# now output path made of URI and name
mgmt_cli -r true show updatable-objects-repository-content filter.text "Amazon"  limit 500 details-level full --format json | jq -r '.objects[] | ."name-in-updatable-objects-repository" as $name | ."additional-properties".uri as $uri | ."uid-in-updatable-objects-repository"  as $uid | ([$uid, $uri, $name] | join("/")) '
```

### Import, list and delete updatable object
Object usable in rule base after adding (import) from repository

```bash
# ssh cp179
#  "9f840200-4376-11e7-bb18-005056c0000c" - Amazon Web Services/S3 Services
mgmt_cli -r true add updatable-object uid-in-updatable-objects-repository "b38fce63-6130-3ea5-98c6-5ad2e9073c7a" --format json

#
mgmt_cli -r true show updatable-objects
# from previous output
mgmt_cli -r true show updatable-object uid  "fea960af-e985-46cb-a3f9-5016bfef68d8" --format json
# delete object
mgmt_cli -r true delete updatable-object uid  "fea960af-e985-46cb-a3f9-5016bfef68d8" --format json
```
