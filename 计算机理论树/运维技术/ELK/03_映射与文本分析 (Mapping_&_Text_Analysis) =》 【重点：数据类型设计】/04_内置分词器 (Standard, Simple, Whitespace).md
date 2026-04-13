
ES 内置了一系列开箱即用的分词器，覆盖基础的文本处理场景，无需额外安装插件，核心内置分词器的规则与适用场景如下：

### 3.4.1 Standard Analyzer（标准分词器，默认分词器）

- **核心规则**：
    
    1. Tokenizer：Standard Tokenizer，按 Unicode 文本边界切分，英文按单词切分，中文、日文、韩文等 CJK 语言按单字切分；
    2. Token Filter：Lowercase Token Filter，将所有词条转换为小写；
    
- **适用场景**：英文等拉丁语系的通用全文检索场景，不适合中文全文检索（单字切分完全丢失中文语义）。
- **分词示例**：
    
    - 原始文本：`The quick brown fox jumps over the lazy dog!`
    - 分词结果：`[the, quick, brown, fox, jumps, over, the, lazy, dog]`
    - 中文原始文本：`我爱中国北京天安门`
    - 中文分词结果：`[我, 爱, 中, 国, 北, 京, 天, 安, 门]`
    

### 3.4.2 Simple Analyzer（简单分词器）

- **核心规则**：
    
    1. Tokenizer：Letter Tokenizer，按非字母字符切分，遇到数字、符号、空格、换行均执行切分；
    2. Token Filter：Lowercase Token Filter，小写转换；
    
- **核心特性**：仅保留纯字母词条，过滤所有数字、符号，适合纯英文文本的简单检索场景。
- **分词示例**：
    
    - 原始文本：`Hello World! 123 Java8 is fun.`
    - 分词结果：`[hello, world, java, is, fun]`
    

### 3.4.3 Whitespace Analyzer（空白符分词器）

- **核心规则**：
    
    1. Tokenizer：Whitespace Tokenizer，仅按空白字符（空格、换行、制表符、回车）切分文本；
    2. Token Filter：无，不做任何额外处理，保留原始大小写与字符；
    
- **核心特性**：不修改原始文本内容，仅按空格切分，适合已提前预处理好的文本、固定格式的关键词文本。
- **分词示例**：
    
    - 原始文本：`Hello World! Java8 123,test`
    - 分词结果：`[Hello, World!, Java8, 123,test]`
    

### 3.4.4 其他常用内置分词器

1. **Keyword Analyzer**：不分词，将整个文本作为单个词条输出，底层使用 Keyword Tokenizer，是 keyword 字段类型的默认分词器；
2. **Stop Analyzer**：在 Simple Analyzer 的基础上，新增 Stop Token Filter，移除英文停用词，适合纯英文的极简检索场景；
3. **Pattern Analyzer**：基于正则表达式自定义切分规则，支持自定义大小写转换、停用词过滤，适配特殊格式的文本处理；
4. **Language Analyzers**：语言专用分词器，支持英语、法语、德语、西班牙语等 30 + 种语言，内置对应语言的词干提取、停用词过滤规则，是对应语言的首选内置分词器。
