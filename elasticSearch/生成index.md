新建一个 Index，指定需要分词的字段。这一步根据数据结构而异，下面的命令只针对本文。基本上，凡是需要搜索的中文字段，都要单独设置一下。

> ```bash
> $ curl -X PUT 'localhost:9200/accounts' -d '
> {
>   "mappings": {
>     "person": {
>       "properties": {
>         "user": {
>           "type": "text",
>           "analyzer": "ik_max_word",
>           "search_analyzer": "ik_max_word"
>         },
>         "title": {
>           "type": "text",
>           "analyzer": "ik_max_word",
>           "search_analyzer": "ik_max_word"
>         },
>         "desc": {
>           "type": "text",
>           "analyzer": "ik_max_word",
>           "search_analyzer": "ik_max_word"
>         }
>       }
>     }
>   }
> }'
> ```

上面代码中，首先新建一个名称为`accounts`的 Index，里面有一个名称为`person`的 Type。`person`有三个字段。

> - user
> - title
> - desc

这三个字段都是中文，而且类型都是文本（text），所以需要指定中文分词器，不能使用默认的英文分词器。

Elastic 的分词器称为 [analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html)。我们对每个字段指定分词器。

> ```javascript
> "user": {
>   "type": "text",
>   "analyzer": "ik_max_word",
>   "search_analyzer": "ik_max_word"
> }
> ```

上面代码中，`analyzer`是字段文本的分词器，`search_analyzer`是搜索词的分词器。`ik_max_word`分词器是插件`ik`提供的，可以对文本进行最大数量的分词。