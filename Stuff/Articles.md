## Kubernetes

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "k8s" AND read = false SORT file.ctime DESC
```

## Container Runtimes

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "container-runtimes" AND read = false SORT file.ctime DESC
```

## Observability

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "observability" AND read = false SORT file.ctime DESC
```

## Security

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "security" AND read = false SORT file.ctime DESC
```

## Network

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "network" AND read = false SORT file.ctime DESC
```

## AWS

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "aws" AND read = false SORT file.ctime DESC
```

## Development

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "development" AND read = false SORT file.ctime DESC
```

## Tools

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "tools" AND read = false SORT file.ctime DESC
```

## Done

```dataview
TABLE WITHOUT IDfile.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source, category as Category, dateformat(file.mtime,"dd/MM/yyyy") as Date FROM "Raw/Notes" WHERE read = true SORT file.mtime DESC
```
