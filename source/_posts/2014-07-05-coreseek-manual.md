---
layout: post
title: "coreseek使用手册"
description: ""
category: "coreseek"
tags: []
---



##产生文本摘要和高亮
### 函数原型: 
`function BuildExcerpts ( $docs, $index, $words, $opts=array() )`
### 相关文档:
([官方文档](http://www.coreseek.cn/docs/coreseek_4.1-sphinx_2.0.1-beta.html#api-func-buildexcerpts))
该函数用来产生文档片段（摘要）。连接到searchd，要求它从指定文档中产生片段（摘要），对命中词高亮，并返回结果。  

* `$docs`为包含各文档内容的数组。必须把需要高亮的文本内容以数组的形式传入到函数中，最后输出的是标红后的文本摘要数组。
*  `$index`为包含索引名字的字符串。给定索引的不同设置（例如字符集、形态学、词形等方面的设置）会被使用。
* `$words`为包含需要高亮的关键字的字符串。它们会按索引的设置被处理。例如，如果英语取词干（stemming）在索引中被设置为允许，那么即使关键词是“shoe”，“shoes”这个词也会被高亮。从版本0.9.9-rc1开始，关键字可以包含通配符，与查询支持的star-syntax类似。
* `$opts`为包含其他可选的高亮参数的hash表：
    + `"before_match"`:在匹配的关键字前面插入的字符串。从版本 1.10-beta开始，可以在该字符串中使用%PASSAGE_ID%宏。该宏会被替换为当前片段的递增值。递增值的起始值默认为1，但是可以通过
    + `"start_passage_id"`设置覆盖。在多文档中调用时，%PASSAGE_ID%会在每篇文档中重新开始。默认为"<b>"。
    + `"after_match"`:在匹配的关键字后面插入的字符串。从版本1.10-beta开始，可以在该字符串中使用%PASSAGE_ID%宏。默认为 "</b>"。
    + `"chunk_separator"`:在摘要块（段落）之间插入的字符串。默认为" ... ".
    + `"limit"`:摘要最多包含的符号（码点）数。整数，默认为256。
    + `"around"`:每个关键词块左右选取的词的数目。整数，默认为 5.
    + `"exact_phrase"`:是否仅高亮精确匹配的整个查询词组，而不是单独的关键词。布尔值，默认为false.
    + `"single_passage"`:是否仅抽取最佳的一个区块。布尔值，默认为false.
    + `"use_boundaries"`:是否跨越由phrase_boundary选项设置的词组边界符。布尔型，默认为false.
    + `"weight_order"`:对于抽取出的段落，要么根据相关度排序（权重下降），要么根据出现在文档中的顺序（位置递增）。布尔型，默认是false.
    + `"query_mode"`:版本1.10-beta新增。设置将`$words`当作 扩展查询语法的查询处理，还是当做普通的文本字符串处理（默认行为）。例如，在查询模式时，("one two" | "three four")仅高亮和包含每个片段中出现"one two" 或 "three four" 的地方及相邻的部分。而在默认模式时， 每个单独出现"one", "two", "three", 或 "four"的地方都会高亮。布尔型，默认是false。
    + `"force_all_words"`:版本1.10-beta新增. 忽略摘要的长度限制直至包含所有的词汇。布尔型，默认为false.
    + `"limit_passages"`:版本1.10-beta新增. 限制摘要中可以包含的最大区块数。整数值，默认为 0 (不限制).
    + `"limit_words"`:版本1.10-beta新增. 限制摘要中可以包含的最大词汇数。整数值，默认为 0 (不限制).
    + `"start_passage_id"`:版本1.10-beta新增. 设置 %PASSAGE_ID% 宏的起始值 (在before_match, after_match 字符串中检查和应用). 整数值，默认为1.
    + `"load_files"`:版本1.10-beta新增. 设置是否将$docs作为摘要抽取来源的数据（默认行为），或者将其当做文件名。从版本2.0.1-beta开始，如果该标志启用，每个请求将创建最多dist_threads个工作线程进行并发处理。布尔型，默认为false.
    + `"html_strip_mode"`:版本1.10-beta新增. HTML标签剥离模式设置。默认为"index"，表示使用index的设置。可以使用的其他值为"none"和"strip"，用于强制跳过或者应用剥离，而不管索引如何设置的。还可以使用"retain"，表示保留HTMK标签并防止高亮时打断标签。"retain"模式仅用于需要高亮整篇文档，并且不能设置限制片段的大小。字符型，可用值为"none"，"strip"，"index"或者"retain"。
    + `"allow_empty"`:版本1.10-beta新增. 允许无法产生摘要时将空字符串作为高亮结果返回 (没有关键字匹配或者不符合片段限制。). 默认情况下，原始文本的开头会被返回而不是空字符串。布尔型，默认为false.
    + `"passage_boundary"`:版本2.0.1-beta新增. 确保区块不跨越句子，段落或者zone区域（仅当每个索引的设置启用时有效）。字符型，可用值为 "sentence", "paragraph", 或者 "zone"."emit_zones":版本2.0.1-beta新增. 在每个区块前使用区域对应的HTML标签来封闭区域。布尔型，魔默认为false。

    摘要提取算法倾向于提取更好的片段（与关键词完全匹配），然后是不在摘要中但是包含了关键词的片段。 通常情况下，它会尝试高亮查询的最佳匹配，并且在限制条件下尽可能的高亮查询中的所有关键词。 如果文档没有匹配查询，默认情况下将根据限制条件返回文档的头部。从版本1.10-beta开始，可以通过设置allow_empty属性位true以返回空的片段来替代默认方式。
失败时返回false。成功时返回包含有片段（摘要）字符串的数组。

### 使用方法
代码片段：
{% codeblock %}
<?php
//...略
include("SphinxClient.php");
class test{


    protected function make_words_highLight($source,$kw){


        $cll = new SphinxClient ();
        $opts = array
            (
                "before_match"      => "<span style='color:red'>",
                "after_match"          => "</span>",
                "chunk_separator"     => " ... ",
                "limit"                => 10,
                "around"               => 6,
            );
        $index = "test1";
        return $cll->BuildExcerpts($source,$index,$kw,$opts);
    }


    //test
    public function test(){
        $content = array("北京是中华人民共和国的首都","中国的首都是北京，2008奥运会就在这里举办的","京东老板刘强东");
        $qsKey = "北京 东";
        $contentArr = $this->make_words_highLight($content,$qsKey);
        print_r($contentArr);die;
    }
}

$m = new test();
$m->test();

?>
//输出结果
Array
(
    [0] => <span style='color:red'>北京</span>是中华人民共和国 ... 
    [1] =>  ... 首都是<span style='color:red'>北京</span>，2008 ... 
    [2] => 京东老板刘强<span style='color:red'>东</span>
)
{% endcodeblock %}
##mmseg 同义词/复合分词处理
###文档([官方文档](http://www.coreseek.cn/opensource/mmseg/#coreseek_mmseg_complex))
mmseg 3.2.13版本开始，提供了类似复合分词的处理方式，供coreseek进行调用。
其基本使用状况为：

* 词库包含：`南京西路、南京、西路`(注意，一定要加入生成uni.lib的词库中)
* 索引时：文本中的“南京西路”会被同时索引为以上三者查
* 查询时：输入南京西路，可以直接匹配南京西路，而不匹配南京或者西路；输入南京或者西路，也可以搜索到南京西路

用法：

1. 处理`unigram.txt`(在`mmseg3/etc`目录下)将词库加入，利用命令`mmseg -u unigram.txt` 生成新的词库`unigram.txt.uni`,用新的词库替代老词库`mv unigram.txt.uni uni.lib`(ps:被替换的同义词必须加入词库中，例如想让一个没有的词`测试`映射同义词`南京西路`则需要`南京西路`在词库中,`测试`不需要加入).
2. 生成同义词库文件mmseg-3.2.13源代码`/script/build_thesaurus.py unigram.txt > thesaurus.txt`(默认的同义词库，可自己修改).
`thesaurus.txt`文件的格式如下:
```
南京西路
-南京,西路,
张三丰
-太极宗师,武当祖师,
```
3. 生成同义词词典:`mmseg -t thesaurus.txt`. 将`thesaurus.lib`放到`uni.lib`同一目录
4. 停止索引服务searchd，重新建立索引，然后启动searchd (ps : 如果只是加入了新的同义词映射，没有修改词表的话不需要重启服务，只需重建索引即可)
5. coreseek索引和搜索时，会自动进行复合分词处理。搜索`南京`，`西路`时都能将搜索`南京西路`的结果匹配到

