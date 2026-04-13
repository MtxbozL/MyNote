
文本分析（Analysis）是全文检索的核心流程，指将非结构化的文本字符串，按照指定规则转换为一系列标准化、可被检索的词条（Term）的过程，也被称为分词过程。

文本分析的核心作用是 **统一索引与查询的文本规范** ：写入文档时，对 text 类型字段执行文本分析，生成的词条构建倒排索引；查询时，对查询文本执行相同规则的文本分析，生成的词条与倒排索引中的词条做匹配，只有经过相同分词规则处理的文本，才能实现精准的全文检索匹配。

### 3.3.1 分词器的三段式核心架构

ES 的分词器（Analyzer）由三个顺序执行的核心模块组成，完整的文本分析流程严格按照**Character Filter  →  Tokenizer  →  Token Filter**的顺序执行，三个模块各司其职，协同完成文本的标准化处理。

```bash
原始文本 → Character Filter 1 → ... → Character Filter N → Tokenizer → Token Filter 1 → ... → Token Filter N → 最终词条列表
```

#### 3.3.1.1 Character Filter（字符过滤器）

Character Filter 是分词器的第一个处理环节，作用于原始文本，核心是对原始字符串做字符级别的转换、过滤、替换处理，处理完成后的文本会传递给 Tokenizer。

一个分词器可以配置 **0 个或多个** Character Filter，按照配置顺序依次执行。ES 内置的核心 Character Filter 包括：

1. **HTML Strip Character Filter**：过滤文本中的 HTML 标签，同时还原 HTML 实体字符，如将`&lt;`转换为`<`，适用于网页内容、富文本内容的处理；
2. **Mapping Character Filter**：自定义字符串映射替换规则，如将敏感词替换为`*`，将繁体中文转换为简体中文；
3. **Pattern Replace Character Filter**：基于正则表达式的字符替换，如去除文本中的特殊符号、数字等。

#### 3.3.1.2 Tokenizer（分词器）

Tokenizer 是分词器的核心必选模块，一个分词器 **有且仅有一个** Tokenizer。其核心作用是将经过 Character Filter 处理后的文本，按照指定规则切分为独立的词条（Token），同时记录每个词条的起始偏移量、位置、长度等元数据，为后续的短语查询、高亮展示提供支撑。

Tokenizer 的核心要求是：必须保证切分后的词条可以还原原始文本的顺序与位置，不能丢失文本的语义关联。ES 内置的核心 Tokenizer 包括：

1. **Standard Tokenizer**：默认分词器，基于 Unicode 文本边界规则切分，对英文按单词切分，对中文按单字切分，支持多语言；
2. **Letter Tokenizer**：按非字母字符切分，遇到数字、符号、空格均切分，将连续的字母作为一个词条；
3. **Whitespace Tokenizer**：仅按空白字符（空格、换行、制表符）切分，不做其他任何处理；
4. **Keyword Tokenizer**：不执行任何切分，将整个文本作为单个词条输出，是 keyword 类型的底层实现。

#### 3.3.1.3 Token Filter（词条过滤器）

Token Filter 是分词器的最后一个处理环节，作用于 Tokenizer 输出的词条列表，核心是对词条做标准化处理，包括大小写转换、停用词移除、同义词扩展、词干提取等，处理完成后的词条列表即为最终的分词结果，用于构建倒排索引。

一个分词器可以配置**0 个或多个**Token Filter，按照配置顺序依次执行，执行顺序会直接影响最终的分词结果。ES 内置的核心 Token Filter 包括：

1. **Lowercase Token Filter**：将所有词条转换为小写，实现大小写不敏感的全文检索，是最常用的 Token Filter；
2. **Stop Token Filter**：移除停用词（无实际检索意义的高频词，如英文的`the`/`a`/`an`，中文的`的`/`了`/`是`），大幅降低倒排索引体积，提升检索性能；
3. **Synonym Token Filter**：同义词扩展，如将`手机`扩展为`手机`/`移动电话`/`智能机`，提升检索召回率；
4. **Stemmer Token Filter**：词干提取，如将英文的`running`/`ran`还原为词干`run`，实现不同时态、语态的单词匹配；
5. **Trim Token Filter**：移除词条首尾的空白字符，避免空格导致的匹配失败。

### 3.3.2 分词器的执行时机

分词器的执行分为两个核心阶段，两个阶段使用的分词器必须保持一致，否则会出现查询匹配失败的问题：

1. **索引时（Index Time）**：文档写入时，对 text 类型的字段执行文本分析，生成的词条用于构建倒排索引，通过字段的`analyzer`参数指定索引时分词器；
2. **查询时（Search Time）**：执行全文检索查询时，对查询文本执行文本分析，生成的词条用于与倒排索引中的词条匹配，通过字段的`search_analyzer`参数指定查询时分词器。

**核心规范**：默认情况下，`search_analyzer`会继承`analyzer`的配置，无需单独指定，保证索引与查询的分词规则完全一致。非特殊场景，严禁单独配置`search_analyzer`，避免分词规则不一致导致的检索异常。

### 3.3.3 自定义分词器

ES 支持基于三段式架构，自定义组合 Character Filter、Tokenizer、Token Filter，实现业务专属的分词器，在索引的`settings`中定义，在`mappings`中为字段指定。

**标准自定义分词器示例**：

```bash
PUT /custom_analyzer_index
{
  "settings": {
    "analysis": {
      "char_filter": {
        // 自定义字符过滤器：去除HTML标签
        "my_html_filter": {
          "type": "html_strip"
        }
      },
      "tokenizer": {
        // 自定义分词器：按标点符号切分
        "my_punctuation_tokenizer": {
          "type": "pattern",
          "pattern": "[.,!?;，。！？；]"
        }
      },
      "filter": {
        // 自定义词条过滤器：小写转换+中文停用词移除
        "my_stop_filter": {
          "type": "stop",
          "stopwords": ["的", "了", "是", "我", "你", "他"]
        }
      },
      "analyzer": {
        // 自定义完整分词器
        "my_custom_analyzer": {
          "type": "custom",
          "char_filter": ["my_html_filter"],
          "tokenizer": "my_punctuation_tokenizer",
          "filter": ["lowercase", "my_stop_filter"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "content": {
        "type": "text",
        "analyzer": "my_custom_analyzer"
      }
    }
  }
}
```
