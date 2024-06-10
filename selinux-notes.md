# SeLinux Notes

## Create a profile based on audit logs

```bash
audit2allow -a -M nested_containers
```

## Enable more verbose logging of events

```bash
semodule -DB
```

## Undo verbose logging

```bash
semodule -B
```

## Display recent denial events

```bash
ausearch -m AVC -ts recent
```

## Apply profile changes

```bash
checkmodule -M -m -o nested_containers.mod nested_containers.te && semodule_package -o nested_containers.pp -m nested_containers.mod && semodule -i nested_containers.pp
```
