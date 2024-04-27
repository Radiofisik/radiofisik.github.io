
```dataview
TABLE file.day, title, description
SORT file.day DESC
```





```dataview
TASK
WHERE !completed
LIMIT 10
GROUP BY file.link
SORT rows.file.ctime ASC
```



```dataview
list from [[]] and !outgoing([[]])
```



```dataview
TABLE file.ctime AS "Created"
WHERE file.ctime >= date(today) - dur(1 week)
```



```dataview
LIST FROM #tag
SORT file.mtime DESC
```
