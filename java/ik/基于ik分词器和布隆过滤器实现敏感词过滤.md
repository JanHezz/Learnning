[TOC]
《基于ik分词器实现敏感词过滤》首发[牧马人博客](http://www.luckyhe.com/post/83.html)转发请加此提示

最近公司业务有个需求，要过滤掉敏感词，涉及到敏感词，我首先就想到了使用分词器以及布隆过滤器来实现它。

## 准备阶段

对市场上分词器进行了一个调研，目前市场上有很多分词器，比如`IkAnalyzer`,`Hanlp`等等。经过一系列的了解我最终选择了'IkAnalyzer'。另外我很快就定位到了`BloomFilter`（布隆过滤器）这是一个`布隆`提出的一个算法。他可以高效以及准确的定位到敏感词是否存在。


#### 为何选用IK Analyzer

`IK Analyzer`是什么呢，一个很轻量的`中文分词工具`，是基于`java`(这是我选用的主要原因)开发的轻量级的中文分词工具包。它是以开源项目Luence为主体的，结合词典分词和文法分析算法的中文分词组件。IK有很多版本，在2012版本中，IK实现了简单的分词歧义排除算法。

这里我们采用了网上的一些介绍。

1、IK才用了特有的“正向迭代最细粒度切分算法”，支持细粒度和智能分词两种切分模式。

2、在系统环境：Core2 i7 3.4G双核，4G内存，window 7 64位， Sun JDK 1.6_29 64位 普通pc环境测试，IK2012具有160万字/秒（3000KB/S）的高速处理能力。

3、2012版的只能分词模式支持简单的分词排歧义处理和数量词合并输出。

4、用了多子处理器分析模式，支持 英文字母 数字 中文词汇等

5、优化词典存储，更小的内存占用。


本人实测

上面效率的问题广大网友已经给出了答案，所以我关注的点事，它分词准不准。

由此我对`IK Analyzer `以及 `Hanlp`进行了一个对比。

>示例:我是一名程序员

`Hanlp`
```java

  long start = System.currentTimeMillis();
        Segment shortestSegment = new DijkstraSegment().enableCustomDictionary(false).enablePlaceRecognize(true).enableOrganizationRecognize(true);

            List<Term> termList =  shortestSegment.seg("我是程序员");

            termList.stream().forEach(term -> {
                System.out.println(term
                .word);

            });

        System.out.println(System.currentTimeMillis()-start);


  分词结果:我，是，程序员。
  耗时:450毫秒

```

`IK Analyzer`
```java

   start = System.currentTimeMillis();
              List<String> result = new ArrayList<>();
        StringReader sr = new StringReader("我是程序员");
        // 关闭智能分词 (对分词的精度影响较大)
        IKSegmenter ik = new IKSegmenter(sr, false);
        Lexeme lex;
        while(true) {
            try {
                if (!((lex=ik.next())!=null)) break;
                   String lexemeText = lex.getLexemeText();
                System.out.println(lexemeText);
                 result.add(lexemeText);
            } catch (IOException e) {
                e.printStackTrace();
            }

        }
        System.out.println(System.currentTimeMillis()-start);

  分词结果:我,是,程序员,程序,员
  耗时：500毫秒

```

结果很显然，`IK Analyzer`的结果更让我满意。效率而言。`IK Analyzer `的弱势可以忽略不计差了`50`毫秒.另外`Ik `支持自定义词汇。比如上面的例子`我是`他是分开的，我们可以使用他的扩展词典功能强行把`我是`变成我们要的词语。具体下文在说。

#### 什么是布隆过滤器

网上一些言论：

布隆过滤器（Bloom Filter）是1970年由布隆提出的。它实际上是一个很长的二进制向量(位图)和一系列随机映射函数（哈希函数）。
布隆过滤器可以用于检索一个元素是否在一个集合中。它的优点是空间效率和查询时间都远远超过一般的算法，缺点是有一定的误识别率和删除困难。

本质上布隆过滤器是一种***数据结构**，比较巧妙的概率型数据结构（probabilistic data structure），特点是高效地插入和查询，可以用来告诉你 “某样东西一定不存在或者可能存在”。

相比于传统的 List、Set、Map 等数据结构，它更高效、占用空间更少，但是缺点是其返回的结果是概率性的，而不是确切的。


抓重点：**数据结构,空间效率和查询时间都远远超过一般的算法**

具体的原理往这看[布隆过滤器原理](https://www.jianshu.com/p/2104d11ee0a2)

## 开发过程

技术选型完毕，这个功能的本质上就是在`布隆过滤器`中使用`IK`f分词，然后判断是否存在敏感词。

#### 整合Ik

pom文件
```xml

<dependency>
            <groupId>com.janeluo</groupId>
            <artifactId>ikanalyzer</artifactId>
            <version>2012_u6</version>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.lucene</groupId>
                    <artifactId>lucene-core</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.lucene</groupId>
                    <artifactId>lucene-queryparser</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.lucene</groupId>
                    <artifactId>lucene-analyzers-common</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <!--  lucene-queryparser 查询分析器模块 -->
        <dependency>
            <groupId>org.apache.lucene</groupId>
            <artifactId>lucene-queryparser</artifactId>
            <version>7.3.0</version>
        </dependency>

```
在resource目录下创建以下文件

IKAnalyzer.cfg.xml (IK Analyzer 扩展配置)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
    <comment>IK Analyzer 扩展配置</comment>
    <entry key="ext_dict">/extend.dic</entry>
    <entry key="ext_stopwords">/stopword.dic</entry>
</properties>
```

extend.dic(扩展字典)
```
我是
你是傻逼
```
这个就是你要扩展的词典，本来`我是`不是一个词语，但是如果你这里扩展了，那么他就会匹配这个词语。

stopword.dic（断点）

```
的
一
```
如果在分词过程中遇到类似关键字会分割。所以我把他解释为断点更为合适。

布隆过滤器结合Ik进行过滤

```java

import com.alibaba.druid.util.StringUtils;
import com.google.api.client.util.Charsets;
import com.google.api.client.util.Lists;
import com.google.common.hash.BloomFilter;
import com.google.common.hash.Funnel;
import com.google.common.hash.PrimitiveSink;
import com.hankcs.hanlp.seg.Dijkstra.DijkstraSegment;
import com.hankcs.hanlp.seg.Segment;
import com.hankcs.hanlp.seg.common.Term;
import lombok.extern.slf4j.Slf4j;
import org.wltea.analyzer.core.IKSegmenter;
import org.wltea.analyzer.core.Lexeme;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.StringReader;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;


/**
 * @CLASSNAME BloomFilterHanlp 布隆过滤器使用hanlp分词
 * @Description
 * @Auther kangning @www.luckyhe.com
 * @DATE 2020/8/20 0020 16:43
 */

@Slf4j
public class BloomFilterHanlp {


    private BloomFilter<String> configuredFilter;

    private final BloomFilter<String> filter = BloomFilter.create(new Funnel<String>() {
        private static final long serialVersionUID = 1L;
        @Override
        public void funnel(String arg0, PrimitiveSink arg1) {

            arg1.putString(arg0, Charsets.UTF_8);
        }
    }, 1024*1024*32);


/**
     * 读取带敏感词的布隆过滤器
     *
     * @return
     * @throws IOException
     */

    public BloomFilter<String> getSensitiveWordsFilter() throws IOException {
        InputStreamReader read = null;
        BufferedReader bufferedReader = null;
        read = new InputStreamReader(this.getClass().getClassLoader().getResourceAsStream("SensitiveWords.txt"), StandardCharsets.UTF_8);
        bufferedReader = new BufferedReader(read);
        for (String txt = null; (txt = bufferedReader.readLine()) != null; ) {
            filter.put(txt);
        }
        this.configuredFilter = filter;
        return filter;
    }


    /**
     * 读取带敏感词的布隆过滤器根据指定关键字
     *
     * @return
     * @throws IOException
     */

    public BloomFilter<String> getSensitiveWordsFilter(List<String> words) throws IOException {

        words.stream().forEach(word->{
            filter.put(word);
        });

        this.configuredFilter = filter;
        return filter;
    }

/**
     * 判断一段文字中，是否包含敏感词
     *
     * @param segment
     * @return
     */

    public Boolean segmentSensitiveFilterPassed(String segment) {
        if(configuredFilter == null){
            try {
                List<String> word=new ArrayList<>(10);
                word.add("程序");
                getSensitiveWordsFilter(word);
            }catch (IOException e){
                e.printStackTrace();
            }
        }
        Segment shortestSegment = new DijkstraSegment().enableCustomDictionary(false).enablePlaceRecognize(true).enableOrganizationRecognize(true);


            List<Term> termList =  shortestSegment.seg(segment);
            for (Term term :termList){
                // 如果布隆过滤器中找到了对应的词，则认为敏感检测不通过
                if(configuredFilter.mightContain(term.word)){
                    log.info("检测到敏感词："+term.word);
                    throw new RuntimeException("检测到敏感词");
                }
            }

        return true;
    }


    /**
     * 判断一段文字中，是否包含敏感词
     *
     * @param segment
     * @return
     */

    public Boolean segmentSensitiveFilterPassedIk(String segment) {
        if(configuredFilter == null){
            try {
                List<String> word=new ArrayList<>(10);
                word.add("程序");
                getSensitiveWordsFilter(word);
            }catch (IOException e){
                e.printStackTrace();
            }
        }
       List<String> result = new ArrayList<>();
        StringReader sr = new StringReader(segment);
        // 关闭智能分词 (对分词的精度影响较大)
        IKSegmenter ik = new IKSegmenter(sr, false);
        Lexeme lex;
        while(true) {
            try {
                if (!((lex=ik.next())!=null)) break;
                   String lexemeText = lex.getLexemeText();
              result.add(lexemeText);
            } catch (IOException e) {
                e.printStackTrace();
            }

        }
            for (String term :result){
                // 如果布隆过滤器中找到了对应的词，则认为敏感检测不通过
                if(configuredFilter.mightContain(term)){
                    log.info("检测到敏感词："+term);
                    throw new RuntimeException("检测到敏感词");
                }
            }
        return true;
    }


    public static void main(String[] args) {

        long start = System.currentTimeMillis();
              List<String> result = new ArrayList<>();
        StringReader sr = new StringReader("我是程序员");
        // 关闭智能分词 (对分词的精度影响较大)
        IKSegmenter ik = new IKSegmenter(sr, false);
        Lexeme lex;
        while(true) {
            try {
                if (!((lex=ik.next())!=null)) break;
                   String lexemeText = lex.getLexemeText();
                System.out.println(lexemeText);
                 result.add(lexemeText);
            } catch (IOException e) {
                e.printStackTrace();
            }

        }
        System.out.println(System.currentTimeMillis()-start);

         BloomFilterHanlp filterHanlp=new BloomFilterHanlp();

        filterHanlp.segmentSensitiveFilterPassedIk("我是程序员");

    }

}

```
#### 总结

关于**布隆过滤器**在敏感词过滤这块有很多应用，但其实`hashMap`也是可以完成这个需求的，`hashMap`的效率也很高的，但是hash有个扩容的问题，如果你的数据量很大，那么不推荐，但是如果你只是很小的数据量。其实`hashMap`也是不错的东西。有兴趣的同学可以对比下。我这边就撤了加班去了淦。