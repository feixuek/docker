# 格式化命令并输出日志

`Docker`使用`Go`模板，可用于处理某些命令和日志驱动程序的输出格式。

`Docker`提供了一组基本功能来操纵模板元素。所有这些示例都使用该`docker inspect`命令，但是许多其他`CLI`命令都具有一个`--format`标志，并且许多`CLI`命令参考都包括自定义输出格式的示例。

## `join`
`join`连接字符串列表以创建单个字符串。它将分隔符放在列表中的每个元素之间。
```shell
docker inspect --format '{{join .Args " , "}}' container
```
## `JSON` 
`json` 将元素编码为`json`字符串。
```shell
docker inspect --format '{{json .Mounts}}' container
```
## `lower`
`lower`将字符串转换为小写形式。
```shell
docker inspect --format "{{lower .Name}}" container
```

## `split`
`split`将字符串切成由分隔符分隔的字符串列表。
```shell
docker inspect --format '{{split .Image ":"}}'
```

## `title`
`title`大写字符串的第一个字符。
```shell
docker inspect --format "{{title .Name}}" container
```

## `upper`
`upper` 将字符串转换为大写形式。
```shell
docker inspect --format "{{upper .Name}}" container
```

## `println` 
`println `在新行上打印每个值。
```shell
docker inspect --format='{{range .NetworkSettings.Networks}}{{println .IPAddress}}{{end}}' container
```

## `Hint`	
要找出可以打印哪些数据，请将所有内容显示为`json`：
```shell
docker container ls --format='{{json .}}'
```