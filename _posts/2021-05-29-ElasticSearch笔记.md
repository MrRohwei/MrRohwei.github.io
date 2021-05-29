[TOC]

# ElasticSearch笔记

## 一、字符过滤器(Char Filter)

> 字符过滤器对未经分析的文本起作用，作用于被分析的文本字段（该字段的index属性为analyzed），字符过滤器在分词器之前工作，用于从文档的原始文本去除HTML标记（markup），或者把字符“&”转换为单词“and”。ElasticSearch 2.4版本内置3个字符过滤器，分别是：映射字符过滤器（Mapping Char Filter）、HTML标记字符过滤器（HTML Strip Char Filter）和模式替换字符过滤器（Pattern Replace Char Filter）。

### 1、映射字符过滤器

> 映射字符过滤器，类型是==**mapping**==，需要建立一个查找字符和替换字符的映射（Mapping），过滤器根据映射把文本中的字符替换成指定的字符。

```json
{
    "index" : {
        "analysis" : {
            "char_filter" : {
                "my_mapping" : {
                    "type" : "mapping",
                    "mappings" : [
                      "ph => f",
                      "qu => k"
                    ]
                }
            },
            "analyzer" : {
                "custom_with_char_filter" : {
                    "tokenizer" : "standard",
                    "char_filter" : ["my_mapping"]
                }
            }
        }
    }
}
```

### 2、HTML标记字符过滤器

> HTML标记字符过滤器，类型是==**html_strip**==，用于从原始文本中去除HTML标记。

### 3、模式替换字符过滤器

> 模式替换字符过滤器，类型是==**pattern_replace**==，它使用正则表达式（Regular Expression）匹配字符，把匹配到的字符替换为指定的替换字符串。

```json
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "tokenizer": "standard",
          "char_filter": [
            "my_char_filter"
          ]
        }
      },
      "char_filter": {
        "my_char_filter": {
          "type": "pattern_replace",
          "pattern": "(\\d+)-(?=\\d)",
          "replacement": "$1_"
        }
      }
    }
  }
}
```

pattern参数：指定Java正则表达式；

replacement参数：指定替换字符串，把正则表达式匹配的字符串替换为replacement参数指定的字符串；

## 二、分词器(Tokenizer)

> 分词器在字符过滤器之后工作，用于把文本分割成多个标记（Token），一个标记基本上是词加上一些额外信息，分词器的处理结果是标记流，它是一个接一个的标记，准备被过滤器处理。

### 1、标准分词器(Standard Tokenizer)

> 标准分词器类型是==**standard**==，用于大多数欧洲语言，使用Unicode文本分割算法对文档进行分词。

### 2、字母分词器(Letter Tokenizer)

> 字符分词器类型是==**letter**==，在非字母位置上分割文本，这就是说，根据相邻的词之间是否存在非字母（例如空格，逗号等）的字符，对文本进行分词，对大多数欧洲语言非常有用。

### 3、空格分词器(Whitespace Tokenizer)

> 空格分词类型是==**whitespace**==，在空格处分割文本

### 4、小写分词器(Lowercase Tokenizer)

> 小写分词器类型是==**lowercase**==，在非字母位置上分割文本，并把分词转换为小写形式，功能上是[Letter Tokenizer](# 2、字母分词器(Letter Tokenizer))和 [Lower Case Token Filter](# 1、小写标记过滤器(Lowercase))的结合（Combination），但是性能更高，一次性完成两个任务。

### 5、经典分词器(Classic Tokenizer)

经典分词器类型是==**classic**==，基于语法规则对文本进行分词，对英语文档分词非常有用，在处理首字母缩写，公司名称，邮件地址和Internet主机名上效果非常好。

## 三、标记过滤器(Token Filter)

> 分析器包含零个或多个标记过滤器，标记过滤器在分词器之后工作，用来处理标记流中的标记。标记过滤从分词器中接收标记流，能够删除标记，转换标记，或添加标记。

### 1、小写标记过滤器(Lowercase)

> 类型是==**lowercase**==，用于把标记转换为小写形式，通过language参数指定语言，小写标记过滤器支持的语言有：Greek, Irish, and Turkish

```yml
index :
    analysis :
        analyzer :
            myAnalyzer2 :
                type : custom
                tokenizer : myTokenizer1
                filter : [myTokenFilter1, myGreekLowerCaseFilter]
                char_filter : [my_html]
        tokenizer :
            myTokenizer1 :
                type : standard
                max_token_length : 900
        filter :
            myTokenFilter1 :
                type : stop
                stopwords : [stop1, stop2, stop3, stop4]
            myGreekLowerCaseFilter :
                type : lowercase
                language : greek
        char_filter :
              my_html :
                type : html_strip
                escaped_tags : [xxx, yyy]
                read_ahead : 1024
```

### 2、停用词标记过滤器(Stopwords)

> 类型是==**stop**==，用于从标记流中移除停用词。参数stopwords用于指定停用词，ElasticSearch 2.4版本提供的预定义的停用词列表：预定义的英语停用词是\_english\_，使用预定义的英语停用词列表是  “stopwords” :"\_english\_"

```json
PUT /my_index
{
    "settings": {
        "analysis": {
            "filter": {
                "my_stop": {
                    "type":       "stop",
                    "stopwords": ["and", "is", "the"]
                }
            }
        }
    }
}
```

### 3、词干过滤器(Stemmer)

> 类型是==**stemmer**==，用于把词转换为其词根形式存储在倒排索引，能够减少标记。

```json
{
    "index" : {
        "analysis" : {
            "analyzer" : {
                "my_analyzer" : {
                    "tokenizer" : "standard",
                    "filter" : ["standard", "lowercase", "my_stemmer"]
                }
            },
            "filter" : {
                "my_stemmer" : {
                    "type" : "stemmer",
                    "name" : "english"
                }
            }
        }
    }
}
```

### 4、同义词过滤器(Synonym)

> 类型是==**synonym**==，在分析阶段，基于同义词规则，把词转换为其同义词存储在倒排索引中

```json
{
    "index" : {
        "analysis" : {
            "analyzer" : {
                "synonym" : {
                    "tokenizer" : "whitespace",
                    "filter" : ["synonym"]
                }
            },
            "filter" : {
                "synonym" : {
                    "type" : "synonym",
                    "synonyms_path" : "analysis/synonym.txt"
                }
            }
        }
    }
}
```

同义词文件的格式实例：

```properties
# Blank lines and lines starting with pound are comments.

# Explicit mappings match any token sequence on the LHS of "=>"
# and replace with all alternatives on the RHS.  These types of mappings
# ignore the expand parameter in the schema.
# Examples:
i-pod, i pod => ipod,
sea biscuit, sea biscit => seabiscuit

# Equivalent synonyms may be separated with commas and give
# no explicit mapping.  In this case the mapping behavior will
# be taken from the expand parameter in the schema.  This allows
# the same synonym file to be used in different synonym handling strategies.
# Examples:
ipod, i-pod, i pod
foozball , foosball
universe , cosmos

# If expand==true, "ipod, i-pod, i pod" is equivalent
# to the explicit mapping:
ipod, i-pod, i pod => ipod, i-pod, i pod
# If expand==false, "ipod, i-pod, i pod" is equivalent
# to the explicit mapping:
ipod, i-pod, i pod => ipod

# Multiple synonym mapping entries are merged.
foo => foo bar
foo => baz
# is equivalent to
foo => foo bar, baz
```

## 四、系统预定义的分析器

在创建索引映射时引用分析器，如果没有定义分析器，那么ElasticSearch将使用默认的分析器，用户可以通过API设置默认的分析器。

default 逻辑名称用于配置在索引和搜索时使用的分析器，default_search 逻辑名称用于配置在搜索时使用的分析器。

```yml
index :
  analysis :
    analyzer :
      default :
        tokenizer : keyword
```

### 1、标准分析器(Standard)

> 分析器类型是==**standard**==，由标准分词器（Standard Tokenizer），标准标记过滤器（Standard Token Filter），小写标记过滤器（Lower Case Token Filter）和停用词标记过滤器（Stopwords Token Filter）组成。参数stopwords用于初始化停用词列表，默认是空的。

### 2、简单分析器(Simple)

> 分析器类型是==**simple**==，实际上是小写标记分词器（Lower Case Tokenizer），在非字母位置上分割文本，并把分词转换为小写形式，功能上是Letter Tokenizer和 Lower Case Token Filter的结合（Combination），但是性能更高，一次性完成两个任务。

### 3、空格分析器(Whitespace)

> 分析器类型是==**whitespace**==，实际上是空格分词器（Whitespace Tokenizer)。

### 4、停用词分析器(Stopwords)

> 分析器类型是==**stop**==，由小写分词器（Lower Case Tokenizer）和停用词标记过滤器（Stop Token Filter）构成，配置参数stopwords 或 stopwords_path指定停用词列表。

### 5、雪球分析器(Snowball)

> 分析器类型是==**snowball**==，由标准分词器（Standard Tokenizer），标准过滤器（Standard Filter），小写过滤器（Lowercase Filter），停用词过滤器（Stop Filter）和雪球过滤器（Snowball Filter）构成。参数language用于指定语言。

```json
{
    "index" : {
        "analysis" : {
            "analyzer" : {
                "my_analyzer" : {
                    "type" : "snowball",
                    "language" : "English"
                }
            }
        }
    }
}
```

### 6、自定义分析器

> 分析器类型是==**custom**==，允许用户定制分析器。参数tokenizer 用于指定分词器，filter用于指定过滤器，char_filter用于指定字符过滤器。

```yaml
index :
    analysis :
        analyzer :
            myAnalyzer2 :
                type : custom
                tokenizer : myTokenizer1
                filter : [myTokenFilter1, myTokenFilter2]
                char_filter : [my_html]
                position_increment_gap: 256
        tokenizer :
            myTokenizer1 :
                type : standard
                max_token_length : 900
        filter :
            myTokenFilter1 :
                type : stop
                stopwords : [stop1, stop2, stop3, stop4]
            myTokenFilter2 :
                type : length
                min : 0
                max : 2000
        char_filter :
              my_html :
                type : html_strip
                escaped_tags : [xxx, yyy]
                read_ahead : 1024
```





















