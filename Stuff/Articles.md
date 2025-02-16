## <mark style="background: #BBFABBA6;">Kubernetes</mark>

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "k8s" AND read = false SORT file.ctime DESC
```

## <mark style="background: #A8CCFF86;">Container Runtimes</mark>

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "container-runtimes" AND read = false SORT file.ctime DESC
```

## <mark style="background: #FF5582A6;">Observability</mark>

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "observability" AND read = false SORT file.ctime DESC
```

## <mark style="background: #ADCCFFA6;">Security</mark>

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "security" AND read = false SORT file.ctime DESC
```

## <mark style="background: #D2B3FFA6;">Network</mark>

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "network" AND read = false SORT file.ctime DESC
```

## <mark style="background: #CACFD9A6;">AWS</mark>

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "aws" AND read = false SORT file.ctime DESC
```

## <mark style="background: #FF5582A6;">Development</mark>

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "development" AND read = false SORT file.ctime DESC
```

## <mark style="background: #FFF3A3A6;">Tools</mark>

```dataview
TABLE WITHOUT ID file.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source FROM "Raw/Notes" WHERE category = "tools" AND read = false SORT file.ctime DESC
```

## <mark style="background: #BBFABBA6;">Done</mark>

```dataview
TABLE WITHOUT IDfile.link AS "Notes", elink(source, split(source, "https?://([\w.\.\-]+)")[1]) as Source, category as Category, dateformat(file.mtime,"dd/MM/yyyy") as Date FROM "Raw/Notes" WHERE read = true SORT file.mtime DESC
```
