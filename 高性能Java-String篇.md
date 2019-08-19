---
title: 高性能Java-String篇
date: 2019-01-04 21:45:12
tags: Java
---
## 前言 ##
字符串处理是程序逻辑中比重比较大的部分，由此带来的资源消耗也是比较多的。在编写代码进行字符串处理时如果能使用一些高效的方法或工具，起码能够帮我们规避一些性能上的坑，避免日后补救。
## 1.字符串拼接 ##
1. 不要用+，虽然在JDK7U40之后编译器会将"+"优化成StringBuilder的方式，但是StringBuilder初始化的时候是不会指定其初始容量的；
2. 用StringBuilder：切记要指定其初始容量，避免扩容造成的CPU和内存浪费，这里造成的浪费还是很可观的。具体内容详见：[StringBuilder你应该知道的几件事情](http://fengfu.io/2018/01/02/StringBuilder%E4%BD%A0%E5%BA%94%E8%AF%A5%E7%9F%A5%E9%81%93%E7%9A%84%E5%87%A0%E4%BB%B6%E4%BA%8B%E6%83%85/)
3. 用Guava Joiner:对于用相同符号间隔的字符串拼接，可以使用Guava的Joiner，用起来很方便。但需要注意的是Joiner.on每次调用都会创建一个新的Joiner实例，会造成内存浪费。同时Joiner是线程安全的，所以对于相同分隔符创建的Joiner实例，公用一个单例就可以啦。
```
  /**
   * Returns a joiner which automatically places {@code separator} between consecutive elements.
   */
  public static Joiner on(char separator) {
    return new Joiner(String.valueOf(separator));
  }
```
## 2.字符串拆分 ##
如果不需要用正则表达式，用StringUtils.split代替String.split，因为原生的split方法支持正则表达式，会导致性能偏低。
![](https://live.staticflickr.com/65535/48288627357_20895e52dd_o.png)
## 3.字符串替换 ##
如果不需要用正则表达式，用StringUtils.replace代替String.replace，因为原生的replace方法支持正则表达式，会导致性能偏低。
```
   /**
     * Replaces each substring of this string that matches the literal target
     * sequence with the specified literal replacement sequence. The
     * replacement proceeds from the beginning of the string to the end, for
     * example, replacing "aa" with "b" in the string "aaa" will result in
     * "ba" rather than "ab".
     *
     * @param  target The sequence of char values to be replaced
     * @param  replacement The replacement sequence of char values
     * @return  The resulting string
     * @since 1.5
     */
    public String replace(CharSequence target, CharSequence replacement) {
        return Pattern.compile(target.toString(), Pattern.LITERAL).matcher(
                this).replaceAll(Matcher.quoteReplacement(replacement.toString()));
    }
```
## 4.字符串转换 ##
避免用String.format。如果你只是想要把一堆不同类型的参数转换成字符串，从性能的角度，建议你直接用StringBuilder实现。因为String.format其实也是用StringBuilder实现的，但由于它要解析format参数中的各种格式进行转换，导致性能降低。有人做过对比，String.format要比直接使用StringBuilder要慢5-30倍……话不多少，直接上代码。
```
    public Formatter format(String format, Object ... args) {
        return format(l, format, args);
    }

    public Formatter format(Locale l, String format, Object ... args) {
        ensureOpen();

        // index of last argument referenced
        int last = -1;
        // last ordinary index
        int lasto = -1;

        FormatString[] fsa = parse(format);
        for (int i = 0; i < fsa.length; i++) {
            FormatString fs = fsa[i];
            int index = fs.index();
            try {
                switch (index) {
                case -2:  // fixed string, "%n", or "%%"
                    fs.print(null, l);
                    break;
                case -1:  // relative index
                    if (last < 0 || (args != null && last > args.length - 1))
                        throw new MissingFormatArgumentException(fs.toString());
                    fs.print((args == null ? null : args[last]), l);
                    break;
                case 0:  // ordinary index
                    lasto++;
                    last = lasto;
                    if (args != null && lasto > args.length - 1)
                        throw new MissingFormatArgumentException(fs.toString());
                    fs.print((args == null ? null : args[lasto]), l);
                    break;
                default:  // explicit index
                    last = index - 1;
                    if (args != null && last > args.length - 1)
                        throw new MissingFormatArgumentException(fs.toString());
                    fs.print((args == null ? null : args[last]), l);
                    break;
                }
            } catch (IOException x) {
                lastException = x;
            }
        }
        return this;
    }
```
上述代码用到了FormatString.print方法，那我们再来看看FormatString的实现：
```
    private interface FormatString {
        int index();
        void print(Object arg, Locale l) throws IOException;
        String toString();
    }

    private class FixedString implements FormatString {
        private String s;
        FixedString(String s) { this.s = s; }
        public int index() { return -2; }
        public void print(Object arg, Locale l)
            throws IOException { a.append(s); }
        public String toString() { return s; }
    }
```
FormatString是Formater的内部接口类，而FixedString实现了FormatString接口，FixedString.print方法用到了a.append()方法，看到append方法，你有没有似曾相识的赶脚呢？我们再来看看这个a是个什么鬼。
```
    public final class Formatter implements Closeable, Flushable {
        private Appendable a;
        private final Locale l;

        private IOException lastException;

        private final char zero;
        private static double scaleUp;

        // 1 (sign) + 19 (max # sig digits) + 1 ('.') + 1 ('e') + 1 (sign)
        // + 3 (max # exp digits) + 4 (error) = 30
        private static final int MAX_FD_CHARS = 30;

        //此处省略一些代码

        private static final Appendable nonNullAppendable(Appendable a) {
            if (a == null)
                return new StringBuilder();

            return a;
        }
        ////此处省略一万字
    }
```
看到了吧，它还是用的StringBuilder，而且没有指定初始化容量，这样如果字符串比较长，扩容带来的资源消耗也是蛮高的。
## 5.toString()方法 ##
toString方法一般是打印日志的时候使用。在这里提一下toString方法的原因是实现toString方法的方式有很多，有手写的、有用ide生成的、有用lombok生成的。这里只提2点：

1. 不建议使用lombok的ToString注解，因为lombok生成的toString方法是用"+"做字符串拼接的，如果打印日志频繁，这里的不必要的性能开销会比较大；
下面的代码是使用了lombok Data和ToString注解的源代码，我们来看看经过lombok处理之后的代码是什么样子。
```
package io.fengfu.learning.lombok;

import lombok.Data;
import lombok.ToString;

import java.util.Date;

@ToString
@Data
public class Wrapper {
    private String wrapperId;
    private boolean isOneWay;
    private String state;
    private int stateCode;
    private String operator;
    private Date date;
    private String operateTime;
    private String detail;
}
```
下面是lombok生成的代码片段：
```
public class Wrapper {
    private String wrapperId;
    private boolean isOneWay;
    private String state;
    private int stateCode;
    private String operator;
    private Date date;
    private String operateTime;
    private String detail;

    public String toString() {
        return "Wrapper(wrapperId=" + getWrapperId() + 
                ", isOneWay=" + isOneWay() + 
                ", state=" + getState() + 
                ", stateCode=" + getStateCode() + 
                ", operator=" + getOperator() + 
                ", date=" + getDate() + 
                ", operateTime=" + 
                getOperateTime() + 
                ", detail=" + 
                getDetail() + ")";
    }
}
```
看到了吗？是用"+"拼接的……

2. 输出格式要统一，这样解析日志时就会方便很多，最起码不用去兼容五花八门的日志格式了。
