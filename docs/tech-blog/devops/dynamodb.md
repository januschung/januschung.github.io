## List all tables

``` bash
aws dynamodb scan list-tables
```

## Backup a table

``` bash
aws dynamodb scan --table-name TABLE-NAME > backup.json
```

## Get a single record by ID

``` bash
aws dynamodb get-item --table-name TABLE-NAME --key '{"id": {"S":"the-id-to-look-for"}}'
```

## Delete a single record by ID

``` bash
aws dynamodb delete-item --table-name TABLE-NAME --key '{"id": {"S":"the-id-to-look-for"}}'
```