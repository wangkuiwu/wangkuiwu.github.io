---
layout: post
title: "Java 字符串系列01 String详解, String和CharSequence区别, StringBuilder和StringBuffer的区别"
description: "java charsequence"
category: java
tags: [java]
date: 2012-04-11 09:01
---

> 本章主要介绍String和CharSequence的区别，以及它们的API详细使用方法。

> **目录**  
[1. String 简介](#anchor1)   
[2. CharSequence和String源码](#anchor2)   
[3. String的API测试代码](#anchor3)   
[4. String 完整示例](#anchor4)   

 
<a name="anchor1"></a>
# 1. String 简介

String 是java中的字符串，它继承于CharSequence。  
String类所包含的API接口非常多。为了便于今后的使用，我对String的API进行了分类，并都给出的演示程序。

**String 和 CharSequence 关系**

String 继承于CharSequence，也就是说String也是CharSequence类型。  
CharSequence是一个接口，它只包括length(), charAt(int index), subSequence(int start, int end)这几个API接口。除了String实现了CharSequence之外，StringBuffer和StringBuilder也实现了CharSequence接口。  
需要说明的是，CharSequence就是字符序列，String, StringBuilder和StringBuffer本质上都是通过字符数组实现的！

 

<br/>
**StringBuilder 和 StringBuffer 的区别**

StringBuilder 和 StringBuffer都是可变的字符序列。它们都继承于AbstractStringBuilder，实现了CharSequence接口。  
但是，StringBuilder是非线程安全的，而StringBuffer是线程安全的。

 

它们之间的关系图如下：

![img](/media/pic/java/charsequence/charsequence-01.jpg)
 
 

**String 函数列表**

    public    String()
    public    String(String original)
    public    String(char[] value)
    public    String(char[] value, int offset, int count)
    public    String(byte[] bytes)
    public    String(byte[] bytes, int offset, int length)
    public    String(byte[] ascii, int hibyte)
    public    String(byte[] ascii, int hibyte, int offset, int count)
    public    String(byte[] bytes, String charsetName)
    public    String(byte[] bytes, int offset, int length, String charsetName)
    public    String(byte[] bytes, Charset charset)
    public    String(byte[] bytes, int offset, int length, Charset charset)
    public    String(int[] codePoints, int offset, int count)
    public    String(StringBuffer buffer)
    public    String(StringBuilder builder)

    public char    charAt(int index)
    public int    codePointAt(int index)
    public int    codePointBefore(int index)
    public int    codePointCount(int beginIndex, int endIndex)
    public int    compareTo(String anotherString)
    public int    compareToIgnoreCase(String str)
    public String    concat(String str)
    public boolean    contains(CharSequence s)
    public boolean    contentEquals(StringBuffer sb)
    public boolean    contentEquals(CharSequence cs)
    public static String    copyValueOf(char[] data, int offset, int count)
    public static String    copyValueOf(char[] data)
    public boolean    endsWith(String suffix)
    public boolean    equals(Object anObject)
    public boolean    equalsIgnoreCase(String anotherString)
    public static String    format(String format, Object[] args)
    public static String    format(Locale l, String format, Object[] args)
    public int    hashCode()
    public int    indexOf(int ch)
    public int    indexOf(int ch, int fromIndex)
    public int    indexOf(String str)
    public int    indexOf(String str, int fromIndex)
    public String    intern()
    public int    lastIndexOf(int ch)
    public int    lastIndexOf(int ch, int fromIndex)
    public int    lastIndexOf(String str)
    public int    lastIndexOf(String str, int fromIndex)
    public int    length()
    public boolean    matches(String regex)
    public int    offsetByCodePoints(int index, int codePointOffset)
    public boolean    regionMatches(int toffset, String other, int ooffset, int len)
    public boolean    regionMatches(boolean ignoreCase, int toffset, String other, int ooffset, int len)
    public String    replace(char oldChar, char newChar)
    public String    replace(CharSequence target, CharSequence replacement)
    public String    replaceAll(String regex, String replacement)
    public String    replaceFirst(String regex, String replacement)
    public String[]    split(String regex, int limit)
    public String[]    split(String regex)
    public boolean    startsWith(String prefix, int toffset)
    public boolean    startsWith(String prefix)
    public CharSequence    subSequence(int beginIndex, int endIndex)
    public String    substring(int beginIndex)
    public String    substring(int beginIndex, int endIndex)
    public char[]    toCharArray()
    public String    toLowerCase(Locale locale)
    public String    toLowerCase()
    public String    toString()
    public String    toUpperCase(Locale locale)
    public String    toUpperCase()
    public String    trim()
    public static String    valueOf(Object obj)
    public static String    valueOf(char[] data)
    public static String    valueOf(char[] data, int offset, int count)
    public static String    valueOf(boolean b)
    public static String    valueOf(char c)
    public static String    valueOf(int i)
    public static String    valueOf(long l)
    public static String    valueOf(float f)
    public static String    valueOf(double d)
    public void    getBytes(int srcBegin, int srcEnd, byte[] dst, int dstBegin)
    public byte[]    getBytes(String charsetName)
    public byte[]    getBytes(Charset charset)
    public byte[]    getBytes()
    public void    getChars(int srcBegin, int srcEnd, char[] dst, int dstBegin)
    public boolean    isEmpty()

 
<a name="anchor2"></a>
# 2. CharSequence和String源码

## 2.1 CharSequence源码(基于jdk1.7.40)

    package java.lang;

    public interface CharSequence {

        int length();

        char charAt(int index);

        CharSequence subSequence(int start, int end);

        public String toString();
    }

## 2.2 String.java源码(基于jdk1.7.40)

    package java.lang;

    import java.io.ObjectStreamField;
    import java.io.UnsupportedEncodingException;
    import java.nio.charset.Charset;
    import java.util.ArrayList;
    import java.util.Arrays;
    import java.util.Comparator;
    import java.util.Formatter;
    import java.util.Locale;
    import java.util.regex.Matcher;
    import java.util.regex.Pattern;
    import java.util.regex.PatternSyntaxException;

    public final class String
        implements java.io.Serializable, Comparable<String>, CharSequence {
        private final char value[];

        private int hash;

        private static final long serialVersionUID = -6849794470754667710L;

        private static final ObjectStreamField[] serialPersistentFields =
                new ObjectStreamField[0];

        public String() {
            this.value = new char[0];
        }

        public String(String original) {
            this.value = original.value;
            this.hash = original.hash;
        }

        public String(char value[]) {
            this.value = Arrays.copyOf(value, value.length);
        }

        public String(char value[], int offset, int count) {
            if (offset < 0) {
                throw new StringIndexOutOfBoundsException(offset);
            }
            if (count < 0) {
                throw new StringIndexOutOfBoundsException(count);
            }
            // Note: offset or count might be near -1>>>1.
            if (offset > value.length - count) {
                throw new StringIndexOutOfBoundsException(offset + count);
            }
            this.value = Arrays.copyOfRange(value, offset, offset+count);
        }

        public String(int[] codePoints, int offset, int count) {
            if (offset < 0) {
                throw new StringIndexOutOfBoundsException(offset);
            }
            if (count < 0) {
                throw new StringIndexOutOfBoundsException(count);
            }
            // Note: offset or count might be near -1>>>1.
            if (offset > codePoints.length - count) {
                throw new StringIndexOutOfBoundsException(offset + count);
            }

            final int end = offset + count;

            // Pass 1: Compute precise size of char[]
            int n = count;
            for (int i = offset; i < end; i++) {
                int c = codePoints[i];
                if (Character.isBmpCodePoint(c))
                    continue;
                else if (Character.isValidCodePoint(c))
                    n++;
                else throw new IllegalArgumentException(Integer.toString(c));
            }

            // Pass 2: Allocate and fill in char[]
            final char[] v = new char[n];

            for (int i = offset, j = 0; i < end; i++, j++) {
                int c = codePoints[i];
                if (Character.isBmpCodePoint(c))
                    v[j] = (char)c;
                else
                    Character.toSurrogates(c, v, j++);
            }

            this.value = v;
        }

        @Deprecated
        public String(byte ascii[], int hibyte, int offset, int count) {
            checkBounds(ascii, offset, count);
            char value[] = new char[count];

            if (hibyte == 0) {
                for (int i = count; i-- > 0;) {
                    value[i] = (char)(ascii[i + offset] & 0xff);
                }
            } else {
                hibyte <<= 8;
                for (int i = count; i-- > 0;) {
                    value[i] = (char)(hibyte | (ascii[i + offset] & 0xff));
                }
            }
            this.value = value;
        }

        @Deprecated
        public String(byte ascii[], int hibyte) {
            this(ascii, hibyte, 0, ascii.length);
        }

         private static void checkBounds(byte[] bytes, int offset, int length) {
            if (length < 0)
                throw new StringIndexOutOfBoundsException(length);
            if (offset < 0)
                throw new StringIndexOutOfBoundsException(offset);
            if (offset > bytes.length - length)
                throw new StringIndexOutOfBoundsException(offset + length);
        }

        public String(byte bytes[], int offset, int length, String charsetName)
                throws UnsupportedEncodingException {
            if (charsetName == null)
                throw new NullPointerException("charsetName");
            checkBounds(bytes, offset, length);
            this.value = StringCoding.decode(charsetName, bytes, offset, length);
        }

        public String(byte bytes[], int offset, int length, Charset charset) {
            if (charset == null)
                throw new NullPointerException("charset");
            checkBounds(bytes, offset, length);
            this.value =  StringCoding.decode(charset, bytes, offset, length);
        }

        public String(byte bytes[], String charsetName)
                throws UnsupportedEncodingException {
            this(bytes, 0, bytes.length, charsetName);
        }

        public String(byte bytes[], Charset charset) {
            this(bytes, 0, bytes.length, charset);
        }

        public String(byte bytes[], int offset, int length) {
            checkBounds(bytes, offset, length);
            this.value = StringCoding.decode(bytes, offset, length);
        }

        public String(byte bytes[]) {
            this(bytes, 0, bytes.length);
        }

        public String(StringBuffer buffer) {
            synchronized(buffer) {
                this.value = Arrays.copyOf(buffer.getValue(), buffer.length());
            }
        }

        public String(StringBuilder builder) {
            this.value = Arrays.copyOf(builder.getValue(), builder.length());
        }

        String(char[] value, boolean share) {
            // assert share : "unshared not supported";
            this.value = value;
        }

        @Deprecated
        String(int offset, int count, char[] value) {
            this(value, offset, count);
        }

        public int length() {
            return value.length;
        }

        public boolean isEmpty() {
            return value.length == 0;
        }

        public char charAt(int index) {
            if ((index < 0) || (index >= value.length)) {
                throw new StringIndexOutOfBoundsException(index);
            }
            return value[index];
        }

        public int codePointAt(int index) {
            if ((index < 0) || (index >= value.length)) {
                throw new StringIndexOutOfBoundsException(index);
            }
            return Character.codePointAtImpl(value, index, value.length);
        }

        public int codePointBefore(int index) {
            int i = index - 1;
            if ((i < 0) || (i >= value.length)) {
                throw new StringIndexOutOfBoundsException(index);
            }
            return Character.codePointBeforeImpl(value, index, 0);
        }

        public int codePointCount(int beginIndex, int endIndex) {
            if (beginIndex < 0 || endIndex > value.length || beginIndex > endIndex) {
                throw new IndexOutOfBoundsException();
            }
            return Character.codePointCountImpl(value, beginIndex, endIndex - beginIndex);
        }

        public int offsetByCodePoints(int index, int codePointOffset) {
            if (index < 0 || index > value.length) {
                throw new IndexOutOfBoundsException();
            }
            return Character.offsetByCodePointsImpl(value, 0, value.length,
                    index, codePointOffset);
        }

        void getChars(char dst[], int dstBegin) {
            System.arraycopy(value, 0, dst, dstBegin, value.length);
        }

        public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {
            if (srcBegin < 0) {
                throw new StringIndexOutOfBoundsException(srcBegin);
            }
            if (srcEnd > value.length) {
                throw new StringIndexOutOfBoundsException(srcEnd);
            }
            if (srcBegin > srcEnd) {
                throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
            }
            System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
        }

        @Deprecated
        public void getBytes(int srcBegin, int srcEnd, byte dst[], int dstBegin) {
            if (srcBegin < 0) {
                throw new StringIndexOutOfBoundsException(srcBegin);
            }
            if (srcEnd > value.length) {
                throw new StringIndexOutOfBoundsException(srcEnd);
            }
            if (srcBegin > srcEnd) {
                throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
            }
            int j = dstBegin;
            int n = srcEnd;
            int i = srcBegin;
            char[] val = value;   /* avoid getfield opcode */

            while (i < n) {
                dst[j++] = (byte)val[i++];
            }
        }

        public byte[] getBytes(String charsetName)
                throws UnsupportedEncodingException {
            if (charsetName == null) throw new NullPointerException();
            return StringCoding.encode(charsetName, value, 0, value.length);
        }

        public byte[] getBytes(Charset charset) {
            if (charset == null) throw new NullPointerException();
            return StringCoding.encode(charset, value, 0, value.length);
        }

        public byte[] getBytes() {
            return StringCoding.encode(value, 0, value.length);
        }

        public boolean equals(Object anObject) {
            if (this == anObject) {
                return true;
            }
            if (anObject instanceof String) {
                String anotherString = (String) anObject;
                int n = value.length;
                if (n == anotherString.value.length) {
                    char v1[] = value;
                    char v2[] = anotherString.value;
                    int i = 0;
                    while (n-- != 0) {
                        if (v1[i] != v2[i])
                                return false;
                        i++;
                    }
                    return true;
                }
            }
            return false;
        }

        public boolean contentEquals(StringBuffer sb) {
            synchronized (sb) {
                return contentEquals((CharSequence) sb);
            }
        }

        public boolean contentEquals(CharSequence cs) {
            if (value.length != cs.length())
                return false;
            // Argument is a StringBuffer, StringBuilder
            if (cs instanceof AbstractStringBuilder) {
                char v1[] = value;
                char v2[] = ((AbstractStringBuilder) cs).getValue();
                int i = 0;
                int n = value.length;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
            // Argument is a String
            if (cs.equals(this))
                return true;
            // Argument is a generic CharSequence
            char v1[] = value;
            int i = 0;
            int n = value.length;
            while (n-- != 0) {
                if (v1[i] != cs.charAt(i))
                    return false;
                i++;
            }
            return true;
        }

        public boolean equalsIgnoreCase(String anotherString) {
            return (this == anotherString) ? true
                    : (anotherString != null)
                    && (anotherString.value.length == value.length)
                    && regionMatches(true, 0, anotherString, 0, value.length);
        }

        public int compareTo(String anotherString) {
            int len1 = value.length;
            int len2 = anotherString.value.length;
            int lim = Math.min(len1, len2);
            char v1[] = value;
            char v2[] = anotherString.value;

            int k = 0;
            while (k < lim) {
                char c1 = v1[k];
                char c2 = v2[k];
                if (c1 != c2) {
                    return c1 - c2;
                }
                k++;
            }
            return len1 - len2;
        }

        public static final Comparator<String> CASE_INSENSITIVE_ORDER
                                             = new CaseInsensitiveComparator();
        private static class CaseInsensitiveComparator
                implements Comparator<String>, java.io.Serializable {
            // use serialVersionUID from JDK 1.2.2 for interoperability
            private static final long serialVersionUID = 8575799808933029326L;

            public int compare(String s1, String s2) {
                int n1 = s1.length();
                int n2 = s2.length();
                int min = Math.min(n1, n2);
                for (int i = 0; i < min; i++) {
                    char c1 = s1.charAt(i);
                    char c2 = s2.charAt(i);
                    if (c1 != c2) {
                        c1 = Character.toUpperCase(c1);
                        c2 = Character.toUpperCase(c2);
                        if (c1 != c2) {
                            c1 = Character.toLowerCase(c1);
                            c2 = Character.toLowerCase(c2);
                            if (c1 != c2) {
                                // No overflow because of numeric promotion
                                return c1 - c2;
                            }
                        }
                    }
                }
                return n1 - n2;
            }
        }

        public int compareToIgnoreCase(String str) {
            return CASE_INSENSITIVE_ORDER.compare(this, str);
        }

        public boolean regionMatches(int toffset, String other, int ooffset,
                int len) {
            char ta[] = value;
            int to = toffset;
            char pa[] = other.value;
            int po = ooffset;
            // Note: toffset, ooffset, or len might be near -1>>>1.
            if ((ooffset < 0) || (toffset < 0)
                    || (toffset > (long)value.length - len)
                    || (ooffset > (long)other.value.length - len)) {
                return false;
            }
            while (len-- > 0) {
                if (ta[to++] != pa[po++]) {
                    return false;
                }
            }
            return true;
        }

        public boolean regionMatches(boolean ignoreCase, int toffset,
                String other, int ooffset, int len) {
            char ta[] = value;
            int to = toffset;
            char pa[] = other.value;
            int po = ooffset;
            // Note: toffset, ooffset, or len might be near -1>>>1.
            if ((ooffset < 0) || (toffset < 0)
                    || (toffset > (long)value.length - len)
                    || (ooffset > (long)other.value.length - len)) {
                return false;
            }
            while (len-- > 0) {
                char c1 = ta[to++];
                char c2 = pa[po++];
                if (c1 == c2) {
                    continue;
                }
                if (ignoreCase) {
                    // If characters don't match but case may be ignored,
                    // try converting both characters to uppercase.
                    // If the results match, then the comparison scan should
                    // continue.
                    char u1 = Character.toUpperCase(c1);
                    char u2 = Character.toUpperCase(c2);
                    if (u1 == u2) {
                        continue;
                    }
                    // Unfortunately, conversion to uppercase does not work properly
                    // for the Georgian alphabet, which has strange rules about case
                    // conversion.  So we need to make one last check before
                    // exiting.
                    if (Character.toLowerCase(u1) == Character.toLowerCase(u2)) {
                        continue;
                    }
                }
                return false;
            }
            return true;
        }

        public boolean startsWith(String prefix, int toffset) {
            char ta[] = value;
            int to = toffset;
            char pa[] = prefix.value;
            int po = 0;
            int pc = prefix.value.length;
            // Note: toffset might be near -1>>>1.
            if ((toffset < 0) || (toffset > value.length - pc)) {
                return false;
            }
            while (--pc >= 0) {
                if (ta[to++] != pa[po++]) {
                    return false;
                }
            }
            return true;
        }

        public boolean startsWith(String prefix) {
            return startsWith(prefix, 0);
        }

        public boolean endsWith(String suffix) {
            return startsWith(suffix, value.length - suffix.value.length);
        }

        public int hashCode() {
            int h = hash;
            if (h == 0 && value.length > 0) {
                char val[] = value;

                for (int i = 0; i < value.length; i++) {
                    h = 31 * h + val[i];
                }
                hash = h;
            }
            return h;
        }

        public int indexOf(int ch) {
            return indexOf(ch, 0);
        }

        public int indexOf(int ch, int fromIndex) {
            final int max = value.length;
            if (fromIndex < 0) {
                fromIndex = 0;
            } else if (fromIndex >= max) {
                // Note: fromIndex might be near -1>>>1.
                return -1;
            }

            if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT) {
                // handle most cases here (ch is a BMP code point or a
                // negative value (invalid code point))
                final char[] value = this.value;
                for (int i = fromIndex; i < max; i++) {
                    if (value[i] == ch) {
                        return i;
                    }
                }
                return -1;
            } else {
                return indexOfSupplementary(ch, fromIndex);
            }
        }

        private int indexOfSupplementary(int ch, int fromIndex) {
            if (Character.isValidCodePoint(ch)) {
                final char[] value = this.value;
                final char hi = Character.highSurrogate(ch);
                final char lo = Character.lowSurrogate(ch);
                final int max = value.length - 1;
                for (int i = fromIndex; i < max; i++) {
                    if (value[i] == hi && value[i + 1] == lo) {
                        return i;
                    }
                }
            }
            return -1;
        }

        public int lastIndexOf(int ch) {
            return lastIndexOf(ch, value.length - 1);
        }

        public int lastIndexOf(int ch, int fromIndex) {
            if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT) {
                // handle most cases here (ch is a BMP code point or a
                // negative value (invalid code point))
                final char[] value = this.value;
                int i = Math.min(fromIndex, value.length - 1);
                for (; i >= 0; i--) {
                    if (value[i] == ch) {
                        return i;
                    }
                }
                return -1;
            } else {
                return lastIndexOfSupplementary(ch, fromIndex);
            }
        }

        private int lastIndexOfSupplementary(int ch, int fromIndex) {
            if (Character.isValidCodePoint(ch)) {
                final char[] value = this.value;
                char hi = Character.highSurrogate(ch);
                char lo = Character.lowSurrogate(ch);
                int i = Math.min(fromIndex, value.length - 2);
                for (; i >= 0; i--) {
                    if (value[i] == hi && value[i + 1] == lo) {
                        return i;
                    }
                }
            }
            return -1;
        }

        public int indexOf(String str) {
            return indexOf(str, 0);
        }

        public int indexOf(String str, int fromIndex) {
            return indexOf(value, 0, value.length,
                    str.value, 0, str.value.length, fromIndex);
        }

        static int indexOf(char[] source, int sourceOffset, int sourceCount,
                char[] target, int targetOffset, int targetCount,
                int fromIndex) {
            if (fromIndex >= sourceCount) {
                return (targetCount == 0 ? sourceCount : -1);
            }
            if (fromIndex < 0) {
                fromIndex = 0;
            }
            if (targetCount == 0) {
                return fromIndex;
            }

            char first = target[targetOffset];
            int max = sourceOffset + (sourceCount - targetCount);

            for (int i = sourceOffset + fromIndex; i <= max; i++) {
                /* Look for first character. */
                if (source[i] != first) {
                    while (++i <= max && source[i] != first);
                }

                /* Found first character, now look at the rest of v2 */
                if (i <= max) {
                    int j = i + 1;
                    int end = j + targetCount - 1;
                    for (int k = targetOffset + 1; j < end && source[j]
                            == target[k]; j++, k++);

                    if (j == end) {
                        /* Found whole string. */
                        return i - sourceOffset;
                    }
                }
            }
            return -1;
        }

        public int lastIndexOf(String str) {
            return lastIndexOf(str, value.length);
        }

        public int lastIndexOf(String str, int fromIndex) {
            return lastIndexOf(value, 0, value.length,
                    str.value, 0, str.value.length, fromIndex);
        }

        static int lastIndexOf(char[] source, int sourceOffset, int sourceCount,
                char[] target, int targetOffset, int targetCount,
                int fromIndex) {
            /*
             * Check arguments; return immediately where possible. For
             * consistency, don't check for null str.
             */
            int rightIndex = sourceCount - targetCount;
            if (fromIndex < 0) {
                return -1;
            }
            if (fromIndex > rightIndex) {
                fromIndex = rightIndex;
            }
            /* Empty string always matches. */
            if (targetCount == 0) {
                return fromIndex;
            }

            int strLastIndex = targetOffset + targetCount - 1;
            char strLastChar = target[strLastIndex];
            int min = sourceOffset + targetCount - 1;
            int i = min + fromIndex;

            startSearchForLastChar:
            while (true) {
                while (i >= min && source[i] != strLastChar) {
                    i--;
                }
                if (i < min) {
                    return -1;
                }
                int j = i - 1;
                int start = j - (targetCount - 1);
                int k = strLastIndex - 1;

                while (j > start) {
                    if (source[j--] != target[k--]) {
                        i--;
                        continue startSearchForLastChar;
                    }
                }
                return start - sourceOffset + 1;
            }
        }

        public String substring(int beginIndex) {
            if (beginIndex < 0) {
                throw new StringIndexOutOfBoundsException(beginIndex);
            }
            int subLen = value.length - beginIndex;
            if (subLen < 0) {
                throw new StringIndexOutOfBoundsException(subLen);
            }
            return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
        }

        public String substring(int beginIndex, int endIndex) {
            if (beginIndex < 0) {
                throw new StringIndexOutOfBoundsException(beginIndex);
            }
            if (endIndex > value.length) {
                throw new StringIndexOutOfBoundsException(endIndex);
            }
            int subLen = endIndex - beginIndex;
            if (subLen < 0) {
                throw new StringIndexOutOfBoundsException(subLen);
            }
            return ((beginIndex == 0) && (endIndex == value.length)) ? this
                    : new String(value, beginIndex, subLen);
        }

        public CharSequence subSequence(int beginIndex, int endIndex) {
            return this.substring(beginIndex, endIndex);
        }

        public String concat(String str) {
            int otherLen = str.length();
            if (otherLen == 0) {
                return this;
            }
            int len = value.length;
            char buf[] = Arrays.copyOf(value, len + otherLen);
            str.getChars(buf, len);
            return new String(buf, true);
        }

        public String replace(char oldChar, char newChar) {
            if (oldChar != newChar) {
                int len = value.length;
                int i = -1;
                char[] val = value; /* avoid getfield opcode */

                while (++i < len) {
                    if (val[i] == oldChar) {
                        break;
                    }
                }
                if (i < len) {
                    char buf[] = new char[len];
                    for (int j = 0; j < i; j++) {
                        buf[j] = val[j];
                    }
                    while (i < len) {
                        char c = val[i];
                        buf[i] = (c == oldChar) ? newChar : c;
                        i++;
                    }
                    return new String(buf, true);
                }
            }
            return this;
        }

        public boolean matches(String regex) {
            return Pattern.matches(regex, this);
        }

        public boolean contains(CharSequence s) {
            return indexOf(s.toString()) > -1;
        }

        public String replaceFirst(String regex, String replacement) {
            return Pattern.compile(regex).matcher(this).replaceFirst(replacement);
        }

        public String replaceAll(String regex, String replacement) {
            return Pattern.compile(regex).matcher(this).replaceAll(replacement);
        }

        public String replace(CharSequence target, CharSequence replacement) {
            return Pattern.compile(target.toString(), Pattern.LITERAL).matcher(
                    this).replaceAll(Matcher.quoteReplacement(replacement.toString()));
        }

        public String[] split(String regex, int limit) {
            /* fastpath if the regex is a
             (1)one-char String and this character is not one of the
                RegEx's meta characters ".$|()[{^?*+\\", or
             (2)two-char String and the first char is the backslash and
                the second is not the ascii digit or ascii letter.
             */
            char ch = 0;
            if (((regex.value.length == 1 &&
                 ".$|()[{^?*+\\".indexOf(ch = regex.charAt(0)) == -1) ||
                 (regex.length() == 2 &&
                  regex.charAt(0) == '\\' &&
                  (((ch = regex.charAt(1))-'0')|('9'-ch)) < 0 &&
                  ((ch-'a')|('z'-ch)) < 0 &&
                  ((ch-'A')|('Z'-ch)) < 0)) &&
                (ch < Character.MIN_HIGH_SURROGATE ||
                 ch > Character.MAX_LOW_SURROGATE))
            {
                int off = 0;
                int next = 0;
                boolean limited = limit > 0;
                ArrayList<String> list = new ArrayList<>();
                while ((next = indexOf(ch, off)) != -1) {
                    if (!limited || list.size() < limit - 1) {
                        list.add(substring(off, next));
                        off = next + 1;
                    } else {    // last one
                        //assert (list.size() == limit - 1);
                        list.add(substring(off, value.length));
                        off = value.length;
                        break;
                    }
                }
                // If no match was found, return this
                if (off == 0)
                    return new String[]{this};

                // Add remaining segment
                if (!limited || list.size() < limit)
                    list.add(substring(off, value.length));

                // Construct result
                int resultSize = list.size();
                if (limit == 0)
                    while (resultSize > 0 && list.get(resultSize - 1).length() == 0)
                        resultSize--;
                String[] result = new String[resultSize];
                return list.subList(0, resultSize).toArray(result);
            }
            return Pattern.compile(regex).split(this, limit);
        }

        public String[] split(String regex) {
            return split(regex, 0);
        }

        public String toLowerCase(Locale locale) {
            if (locale == null) {
                throw new NullPointerException();
            }

            int firstUpper;
            final int len = value.length;

            /* Now check if there are any characters that need to be changed. */
            scan: {
                for (firstUpper = 0 ; firstUpper < len; ) {
                    char c = value[firstUpper];
                    if ((c >= Character.MIN_HIGH_SURROGATE)
                            && (c <= Character.MAX_HIGH_SURROGATE)) {
                        int supplChar = codePointAt(firstUpper);
                        if (supplChar != Character.toLowerCase(supplChar)) {
                            break scan;
                        }
                        firstUpper += Character.charCount(supplChar);
                    } else {
                        if (c != Character.toLowerCase(c)) {
                            break scan;
                        }
                        firstUpper++;
                    }
                }
                return this;
            }

            char[] result = new char[len];
            int resultOffset = 0;  /* result may grow, so i+resultOffset
                                    * is the write location in result */

            /* Just copy the first few lowerCase characters. */
            System.arraycopy(value, 0, result, 0, firstUpper);

            String lang = locale.getLanguage();
            boolean localeDependent =
                    (lang == "tr" || lang == "az" || lang == "lt");
            char[] lowerCharArray;
            int lowerChar;
            int srcChar;
            int srcCount;
            for (int i = firstUpper; i < len; i += srcCount) {
                srcChar = (int)value[i];
                if ((char)srcChar >= Character.MIN_HIGH_SURROGATE
                        && (char)srcChar <= Character.MAX_HIGH_SURROGATE) {
                    srcChar = codePointAt(i);
                    srcCount = Character.charCount(srcChar);
                } else {
                    srcCount = 1;
                }
                if (localeDependent || srcChar == '\u03A3') { // GREEK CAPITAL LETTER SIGMA
                    lowerChar = ConditionalSpecialCasing.toLowerCaseEx(this, i, locale);
                } else if (srcChar == '\u0130') { // LATIN CAPITAL LETTER I DOT
                    lowerChar = Character.ERROR;
                } else {
                    lowerChar = Character.toLowerCase(srcChar);
                }
                if ((lowerChar == Character.ERROR)
                        || (lowerChar >= Character.MIN_SUPPLEMENTARY_CODE_POINT)) {
                    if (lowerChar == Character.ERROR) {
                        if (!localeDependent && srcChar == '\u0130') {
                            lowerCharArray =
                                    ConditionalSpecialCasing.toLowerCaseCharArray(this, i, Locale.ENGLISH);
                        } else {
                            lowerCharArray =
                                    ConditionalSpecialCasing.toLowerCaseCharArray(this, i, locale);
                        }
                    } else if (srcCount == 2) {
                        resultOffset += Character.toChars(lowerChar, result, i + resultOffset) - srcCount;
                        continue;
                    } else {
                        lowerCharArray = Character.toChars(lowerChar);
                    }

                    /* Grow result if needed */
                    int mapLen = lowerCharArray.length;
                    if (mapLen > srcCount) {
                        char[] result2 = new char[result.length + mapLen - srcCount];
                        System.arraycopy(result, 0, result2, 0, i + resultOffset);
                        result = result2;
                    }
                    for (int x = 0; x < mapLen; ++x) {
                        result[i + resultOffset + x] = lowerCharArray[x];
                    }
                    resultOffset += (mapLen - srcCount);
                } else {
                    result[i + resultOffset] = (char)lowerChar;
                }
            }
            return new String(result, 0, len + resultOffset);
        }

        public String toLowerCase() {
            return toLowerCase(Locale.getDefault());
        }

        public String toUpperCase(Locale locale) {
            if (locale == null) {
                throw new NullPointerException();
            }

            int firstLower;
            final int len = value.length;

            /* Now check if there are any characters that need to be changed. */
            scan: {
               for (firstLower = 0 ; firstLower < len; ) {
                    int c = (int)value[firstLower];
                    int srcCount;
                    if ((c >= Character.MIN_HIGH_SURROGATE)
                            && (c <= Character.MAX_HIGH_SURROGATE)) {
                        c = codePointAt(firstLower);
                        srcCount = Character.charCount(c);
                    } else {
                        srcCount = 1;
                    }
                    int upperCaseChar = Character.toUpperCaseEx(c);
                    if ((upperCaseChar == Character.ERROR)
                            || (c != upperCaseChar)) {
                        break scan;
                    }
                    firstLower += srcCount;
                }
                return this;
            }

            char[] result = new char[len]; /* may grow */
            int resultOffset = 0;  /* result may grow, so i+resultOffset
             * is the write location in result */

            /* Just copy the first few upperCase characters. */
            System.arraycopy(value, 0, result, 0, firstLower);

            String lang = locale.getLanguage();
            boolean localeDependent =
                    (lang == "tr" || lang == "az" || lang == "lt");
            char[] upperCharArray;
            int upperChar;
            int srcChar;
            int srcCount;
            for (int i = firstLower; i < len; i += srcCount) {
                srcChar = (int)value[i];
                if ((char)srcChar >= Character.MIN_HIGH_SURROGATE &&
                    (char)srcChar <= Character.MAX_HIGH_SURROGATE) {
                    srcChar = codePointAt(i);
                    srcCount = Character.charCount(srcChar);
                } else {
                    srcCount = 1;
                }
                if (localeDependent) {
                    upperChar = ConditionalSpecialCasing.toUpperCaseEx(this, i, locale);
                } else {
                    upperChar = Character.toUpperCaseEx(srcChar);
                }
                if ((upperChar == Character.ERROR)
                        || (upperChar >= Character.MIN_SUPPLEMENTARY_CODE_POINT)) {
                    if (upperChar == Character.ERROR) {
                        if (localeDependent) {
                            upperCharArray =
                                    ConditionalSpecialCasing.toUpperCaseCharArray(this, i, locale);
                        } else {
                            upperCharArray = Character.toUpperCaseCharArray(srcChar);
                        }
                    } else if (srcCount == 2) {
                        resultOffset += Character.toChars(upperChar, result, i + resultOffset) - srcCount;
                        continue;
                    } else {
                        upperCharArray = Character.toChars(upperChar);
                    }

                    /* Grow result if needed */
                    int mapLen = upperCharArray.length;
                    if (mapLen > srcCount) {
                        char[] result2 = new char[result.length + mapLen - srcCount];
                        System.arraycopy(result, 0, result2, 0, i + resultOffset);
                        result = result2;
                    }
                    for (int x = 0; x < mapLen; ++x) {
                        result[i + resultOffset + x] = upperCharArray[x];
                    }
                    resultOffset += (mapLen - srcCount);
                } else {
                    result[i + resultOffset] = (char)upperChar;
                }
            }
            return new String(result, 0, len + resultOffset);
        }

        public String toUpperCase() {
            return toUpperCase(Locale.getDefault());
        }

        public String trim() {
            int len = value.length;
            int st = 0;
            char[] val = value;    /* avoid getfield opcode */

            while ((st < len) && (val[st] <= ' ')) {
                st++;
            }
            while ((st < len) && (val[len - 1] <= ' ')) {
                len--;
            }
            return ((st > 0) || (len < value.length)) ? substring(st, len) : this;
        }

        public String toString() {
            return this;
        }

        public char[] toCharArray() {
            // Cannot use Arrays.copyOf because of class initialization order issues
            char result[] = new char[value.length];
            System.arraycopy(value, 0, result, 0, value.length);
            return result;
        }

        public static String format(String format, Object... args) {
            return new Formatter().format(format, args).toString();
        }

        public static String format(Locale l, String format, Object... args) {
            return new Formatter(l).format(format, args).toString();
        }

        public static String valueOf(Object obj) {
            return (obj == null) ? "null" : obj.toString();
        }

        public static String valueOf(char data[]) {
            return new String(data);
        }

        public static String valueOf(char data[], int offset, int count) {
            return new String(data, offset, count);
        }

        public static String copyValueOf(char data[], int offset, int count) {
            // All public String constructors now copy the data.
            return new String(data, offset, count);
        }

        public static String copyValueOf(char data[]) {
            return new String(data);
        }

        public static String valueOf(boolean b) {
            return b ? "true" : "false";
        }

        public static String valueOf(char c) {
            char data[] = {c};
            return new String(data, true);
        }

        public static String valueOf(int i) {
            return Integer.toString(i);
        }

        public static String valueOf(long l) {
            return Long.toString(l);
        }

        public static String valueOf(float f) {
            return Float.toString(f);
        }

        public static String valueOf(double d) {
            return Double.toString(d);
        }

        public native String intern();

        private static final int HASHING_SEED;

        static {
            long nanos = System.nanoTime();
            long now = System.currentTimeMillis();
            int SEED_MATERIAL[] = {
                    System.identityHashCode(String.class),
                    System.identityHashCode(System.class),
                    (int) (nanos >>> 32),
                    (int) nanos,
                    (int) (now >>> 32),
                    (int) now,
                    (int) (System.nanoTime() >>> 2)
            };

            // Use murmur3 to scramble the seeding material.
            // Inline implementation to avoid loading classes
            int h1 = 0;

            // body
            for (int k1 : SEED_MATERIAL) {
                k1 *= 0xcc9e2d51;
                k1 = (k1 << 15) | (k1 >>> 17);
                k1 *= 0x1b873593;

                h1 ^= k1;
                h1 = (h1 << 13) | (h1 >>> 19);
                h1 = h1 * 5 + 0xe6546b64;
            }

            // tail (always empty, as body is always 32-bit chunks)

            // finalization

            h1 ^= SEED_MATERIAL.length * 4;

            // finalization mix force all bits of a hash block to avalanche
            h1 ^= h1 >>> 16;
            h1 *= 0x85ebca6b;
            h1 ^= h1 >>> 13;
            h1 *= 0xc2b2ae35;
            h1 ^= h1 >>> 16;

            HASHING_SEED = h1;
        }

        private transient int hash32 = 0;


        int hash32() {
            int h = hash32;
            if (0 == h) {
               // harmless data race on hash32 here.
               h = sun.misc.Hashing.murmur3_32(HASHING_SEED, value, 0, value.length);

               // ensure result is not zero to avoid recalcing
               h = (0 != h) ? h : 1;

               hash32 = h;
            }

            return h;
        }
    }

说明：String的本质是字符序列，它是通过字符数组实现的！

 

<a name="anchor3"></a>
# 3. String的API测试代码

## 3.1 CharSequence

下面通过示例，演示CharSequence的使用方法！  
源码如下(CharSequenceTest.java):

    /**
     * CharSequence 演示程序
     *
     * @author skywang
     */
    import java.nio.charset.Charset;
    import java.io.UnsupportedEncodingException;

    public class CharSequenceTest {

        public static void main(String[] args) {
            testCharSequence();
        }

        /**
         * CharSequence 测试程序
         */
        private static void testCharSequence() {
            System.out.println("-------------------------------- testCharSequence -----------------------------");

            // 1. CharSequence的子类String
            String str = "abcdefghijklmnopqrstuvwxyz";
            System.out.println("1. String");
            System.out.printf("   %-30s=%d\n", "str.length()", str.length());
            System.out.printf("   %-30s=%c\n", "str.charAt(5)", str.charAt(5));
            String substr = (String)str.subSequence(0,5);
            System.out.printf("   %-30s=%s\n", "str.subSequence(0,5)", substr.toString());

            // 2. CharSequence的子类StringBuilder
            StringBuilder strbuilder = new StringBuilder("abcdefghijklmnopqrstuvwxyz");
            System.out.println("2. StringBuilder");
            System.out.printf("   %-30s=%d\n", "strbuilder.length()", strbuilder.length());
            System.out.printf("   %-30s=%c\n", "strbuilder.charAt(5)", strbuilder.charAt(5));
            // 注意：StringBuilder的subSequence()返回的是，实际上是一个String对象！
            String substrbuilder = (String)strbuilder.subSequence(0,5);
            System.out.printf("   %-30s=%s\n", "strbuilder.subSequence(0,5)", substrbuilder.toString());

            // 3. CharSequence的子类StringBuffer
            StringBuffer strbuffer = new StringBuffer("abcdefghijklmnopqrstuvwxyz");
            System.out.println("3. StringBuffer");
            System.out.printf("   %-30s=%d\n", "strbuffer.length()", strbuffer.length());
            System.out.printf("   %-30s=%c\n", "strbuffer.charAt(5)", strbuffer.charAt(5));
            // 注意：StringBuffer的subSequence()返回的是，实际上是一个String对象！
            String substrbuffer = (String)strbuffer.subSequence(0,5);
            System.out.printf("   %-30s=%s\n", "strbuffer.subSequence(0,5)", substrbuffer.toString());

            System.out.println();
        }
    }

运行结果：

    -------------------------------- testCharSequence -----------------------------
    1. String
       str.length()                  =26
       str.charAt(5)                 =f
       str.subSequence(0,5)          =abcde
    2. StringBuilder
       strbuilder.length()           =26
       strbuilder.charAt(5)          =f
       strbuilder.subSequence(0,5)   =abcde
    3. StringBuffer
       strbuffer.length()            =26
       strbuffer.charAt(5)           =f
       strbuffer.subSequence(0,5)    =abcde

 
## 3.2 String 构造函数

下面通过示例，演示String的各种构造函数的使用方法！  
源码如下(StringContructorTest.java):

    /**
     * String 构造函数演示程序
     *
     * @author skywang
     */
    import java.nio.charset.Charset;
    import java.io.UnsupportedEncodingException;

    public class StringContructorTest {

        public static void main(String[] args) {
            testStringConstructors() ;
        }

        /**
         * String 构造函数测试程序
         */
        private static void testStringConstructors() {
            try {
                System.out.println("-------------------------------- testStringConstructors -----------------------");

                String str01 = new String();
                String str02 = new String("String02");
                String str03 = new String(new char[]{'s','t','r','0','3'});
                String str04 = new String(new char[]{'s','t','r','0','4'}, 1, 3);          // 1表示起始位置，3表示个数
                String str05 = new String(new byte[]{0x61, 0x62, 0x63, 0x64, 0x65});       // 0x61在ASC表中，对应字符"a"; 1表示起始位置，3表示长度
                String str06 = new String(new byte[]{0x61, 0x62, 0x63, 0x64, 0x65}, 1, 3); // 0x61在ASC表中，对应字符"a"; 1表示起始位置，3表示长度
                String str07 = new String(new byte[]{0x61, 0x62, 0x63, 0x64, 0x65}, 0);       // 0x61在ASC表中，对应字符"a";0，表示“高字节”
                String str08 = new String(new byte[]{0x61, 0x62, 0x63, 0x64, 0x65}, 0, 1, 3); // 0x61在ASC表中，对应字符"a"; 0，表示“高字节”；1表示起始位置，3表示长度
                String str09 = new String(new byte[]{(byte)0xe5, (byte)0xad, (byte)0x97, /* 字-对应的utf-8编码 */ 
                                                     (byte)0xe7, (byte)0xac, (byte)0xa6, /* 符-对应的utf-8编码 */ 
                                                     (byte)0xe7, (byte)0xbc, (byte)0x96, /* 编-对应的utf-8编码 */ 
                                                     (byte)0xe7, (byte)0xa0, (byte)0x81, /* 码-对应的utf-8编码 */ }, 
                                          0, 12, "utf-8");  // 0表示起始位置，12表示长度。
                String str10 = new String(new byte[]{(byte)0x5b, (byte)0x57, /* 字-对应的utf-16编码 */ 
                                                     (byte)0x7b, (byte)0x26, /* 符-对应的utf-16编码 */ 
                                                     (byte)0x7f, (byte)0x16, /* 编-对应的utf-16编码 */ 
                                                     (byte)0x78, (byte)0x01, /* 码-对应的utf-16编码 */ }, 
                                          0, 8, "utf-16");  // 0表示起始位置，8表示长度。
                String str11 = new String(new byte[]{(byte)0xd7, (byte)0xd6, /* 字-对应的gb2312编码  */ 
                                                     (byte)0xb7, (byte)0xfb, /* 符-对应的gb2312编码 */ 
                                                     (byte)0xb1, (byte)0xe0, /* 编-对应的gb2312编码 */ 
                                                     (byte)0xc2, (byte)0xeb, /* 码-对应的gb2312编码 */ }, 
                                          Charset.forName("gb2312")); 
                String str12 = new String(new byte[]{(byte)0xd7, (byte)0xd6, /* 字-对应的gbk编码 */ 
                                                     (byte)0xb7, (byte)0xfb, /* 符-对应的gbk编码 */ 
                                                     (byte)0xb1, (byte)0xe0, /* 编-对应的gbk编码 */ 
                                                     (byte)0xc2, (byte)0xeb, /* 码-对应的gbk编码 */ }, 
                                          0, 8, Charset.forName("gbk")); 
                String str13 = new String(new int[] {0x5b57, 0x7b26, 0x7f16, 0x7801}, 0, 4);  // "字符编码"(\u5b57是‘字’的unicode编码)。0表示起始位置，4表示长度。
                String str14 = new String(new StringBuffer("StringBuffer"));
                String str15 = new String(new StringBuilder("StringBuilder"));

                System.out.printf(" str01=%s \n str02=%s \n str03=%s \n str04=%s \n str05=%s \n str06=%s \n str07=%s \n str08=%s\n str09=%s\n str10=%s\n str11=%s\n str12=%s\n str13=%s\n str14=%s\n str15=%s\n",
                        str01, str02, str03, str04, str05, str06, str07, str08, str09, str10, str11, str12, str13, str14, str15);


                System.out.println();
            } catch (UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        }
    }

运行结果：

    -------------------------------- testStringConstructors -----------------------
     str01= 
     str02=String02 
     str03=str03 
     str04=tr0 
     str05=abcde 
     str06=bcd 
     str07=abcde 
     str08=bcd
     str09=字符编码
     str10=字符编码
     str11=字符编码
     str12=字符编码
     str13=字符编码
     str14=StringBuffer
     str15=StringBuilder

 
## 3.3 String 将各种对象转换成String的API

源码如下(StringValueTest.java):

    /**
     * String value相关示例
     *
     * @author skywang
     */
    import java.util.HashMap;

    public class StringValueTest {
        
        public static void main(String[] args) {
            testValueAPIs() ;
        }

        /**
         * String 的valueOf()演示程序
         */
        private static void testValueAPIs() {
            System.out.println("-------------------------------- testValueAPIs --------------------------------");
            // 1. String    valueOf(Object obj)
            //  实际上，返回的是obj.toString();
            HashMap map = new HashMap();
            map.put("1", "one");
            map.put("2", "two");
            map.put("3", "three");
            System.out.printf("%-50s = %s\n", "String.valueOf(map)", String.valueOf(map));

            // 2.String    valueOf(boolean b)
            System.out.printf("%-50s = %s\n", "String.valueOf(true)", String.valueOf(true));

            // 3.String    valueOf(char c)
            System.out.printf("%-50s = %s\n", "String.valueOf('m')", String.valueOf('m'));

            // 4.String    valueOf(int i)
            System.out.printf("%-50s = %s\n", "String.valueOf(96)", String.valueOf(96));

            // 5.String    valueOf(long l)
            System.out.printf("%-50s = %s\n", "String.valueOf(12345L)", String.valueOf(12345L));

            // 6.String    valueOf(float f)
            System.out.printf("%-50s = %s\n", "String.valueOf(1.414f)", String.valueOf(1.414f));

            // 7.String    valueOf(double d)
            System.out.printf("%-50s = %s\n", "String.valueOf(3.14159d)", String.valueOf(3.14159d));

            // 8.String    valueOf(char[] data)
            System.out.printf("%-50s = %s\n", "String.valueOf(new char[]{'s','k','y'})", String.valueOf(new char[]{'s','k','y'}));

            // 9.String    valueOf(char[] data, int offset, int count)
            System.out.printf("%-50s = %s\n", "String.valueOf(new char[]{'s','k','y'}, 0, 2)", String.valueOf(new char[]{'s','k','y'}, 0, 2));

            System.out.println();
        }
    }

运行结果：

    -------------------------------- testValueAPIs --------------------------------
    String.valueOf(map)                                = {3=three, 2=two, 1=one}
    String.valueOf(true)                               = true
    String.valueOf('m')                                = m
    String.valueOf(96)                                 = 96
    String.valueOf(12345L)                             = 12345
    String.valueOf(1.414f)                             = 1.414
    String.valueOf(3.14159d)                           = 3.14159
    String.valueOf(new char[]{'s','k','y'})            = sky
    String.valueOf(new char[]{'s','k','y'}, 0, 2)      = sk

 
## 3.4 String 中index相关的API

源码如下(StringIndexTest.java):

    /**
     * String 中index相关API演示
     *
     * @author skywang
     */

    public class StringIndexTest {

        public static void main(String[] args) {
            testIndexAPIs() ;
        }

        /**
         * String 中index相关API演示
         */
        private static void testIndexAPIs() {
            System.out.println("-------------------------------- testIndexAPIs --------------------------------");

            String istr = "abcAbcABCabCaBcAbCaBCabc";
            System.out.printf("istr=%s\n", istr);

            // 1. 从前往后，找出‘a’第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.indexOf((int)'a')", istr.indexOf((int)'a'));

            // 2. 从位置5开始，从前往后，找出‘a’第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.indexOf((int)'a', 5)", istr.indexOf((int)'a', 5));

            // 3. 从后往前，找出‘a’第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.lastIndexOf((int)'a')", istr.lastIndexOf((int)'a'));

            // 4. 从位置10开始，从后往前，找出‘a’第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.lastIndexOf((int)'a', 10)", istr.lastIndexOf((int)'a', 10));


            // 5. 从前往后，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.indexOf(\"bc\")", istr.indexOf("bc"));

            // 6. 从位置5开始，从前往后，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.indexOf(\"bc\", 5)", istr.indexOf("bc", 5));

            // 7. 从后往前，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.lastIndexOf(\"bc\")", istr.lastIndexOf("bc"));

            // 8. 从位置4开始，从后往前，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.lastIndexOf(\"bc\", 4)", istr.lastIndexOf("bc", 4));

            System.out.println();
        }
    }

运行结果：

    -------------------------------- testIndexAPIs --------------------------------
    istr=abcAbcABCabCaBcAbCaBCabc
    istr.indexOf((int)'a')         = 0
    istr.indexOf((int)'a', 5)      = 9
    istr.lastIndexOf((int)'a')     = 21
    istr.lastIndexOf((int)'a', 10) = 9
    istr.indexOf("bc")             = 1
    istr.indexOf("bc", 5)          = 22
    istr.lastIndexOf("bc")         = 22
    istr.lastIndexOf("bc", 4)      = 4

 
## 3.5 String “比较”操作的API

源码如下(StringCompareTest.java):

    /**
     * String 中比较相关API演示
     *
     * @author skywang
     */

    public class StringCompareTest {

        public static void main(String[] args) {
            testCompareAPIs() ;
        }

        /**
         * String 中比较相关API演示
         */
        private static void testCompareAPIs() {
            System.out.println("-------------------------------- testCompareAPIs ------------------------------");

            //String str = "abcdefghijklmnopqrstuvwxyz";
            String str = "abcAbcABCabCAbCabc";
            System.out.printf("str=%s\n", str);

            // 1. 比较“2个String是否相等”
            System.out.printf("%-50s = %b\n", 
                    "str.equals(\"abcAbcABCabCAbCabc\")", 
                    str.equals("abcAbcABCabCAbCabc"));

            // 2. 比较“2个String是否相等(忽略大小写)”
            System.out.printf("%-50s = %b\n", 
                    "str.equalsIgnoreCase(\"ABCABCABCABCABCABC\")", 
                    str.equalsIgnoreCase("ABCABCABCABCABCABC"));

            // 3. 比较“2个String的大小”
            System.out.printf("%-40s = %d\n", "str.compareTo(\"abce\")", str.compareTo("abce"));

            // 4. 比较“2个String的大小(忽略大小写)”
            System.out.printf("%-40s = %d\n", "str.compareToIgnoreCase(\"ABC\")", str.compareToIgnoreCase("ABC"));

            // 5. 字符串的开头是不是"ab"
            System.out.printf("%-40s = %b\n", "str.startsWith(\"ab\")", str.startsWith("ab"));

            // 6. 字符串的从位置3开头是不是"ab"
            System.out.printf("%-40s = %b\n", "str.startsWith(\"Ab\")", str.startsWith("Ab", 3));

            // 7. 字符串的结尾是不是"bc"
            System.out.printf("%-40s = %b\n", "str.endsWith(\"bc\")", str.endsWith("bc"));

            // 8. 字符串的是不是包含"ABC"
            System.out.printf("%-40s = %b\n", "str.contains(\"ABC\")", str.contains("ABC"));

            // 9. 比较2个字符串的部分内容
            String region1 = str.substring(2, str.length());    // 获取str位置3(包括)到末尾(不包括)的子字符串
            // 将“str中从位置2开始的字符串”和“region1中位置0开始的字符串”进行比较，比较长度是5。
            System.out.printf("regionMatches(%s) = %b\n", region1, 
                    str.regionMatches(2, region1, 0, 5));

            // 10. 比较2个字符串的部分内容(忽略大小写)
            String region2 = region1.toUpperCase();    // 将region1转换为大写
            String region3 = region1.toLowerCase();    // 将region1转换为小写
            System.out.printf("regionMatches(%s) = %b\n", region2, 
                    str.regionMatches(2, region2, 0, 5));
            System.out.printf("regionMatches(%s) = %b\n", region3, 
                    str.regionMatches(2, region3, 0, 5));

            // 11. 比较“String”和“StringBuffer”的内容是否相等
            System.out.printf("%-60s = %b\n", 
                    "str.contentEquals(new StringBuffer(\"abcAbcABCabCAbCabc\"))", 
                    str.contentEquals(new StringBuffer("abcAbcABCabCAbCabc")));

            // 12. 比较“String”和“StringBuilder”的内容是否相等
            System.out.printf("%-60s = %b\n", 
                    "str.contentEquals(new StringBuilder(\"abcAbcABCabCAbCabc\"))", 
                    str.contentEquals(new StringBuilder("abcAbcABCabCAbCabc")));

            // 13. match()测试程序
            // 正则表达式 xxx.xxx.xxx.xxx，其中xxx中x的取值可以是0～9，xxx中有1～3位。
            String reg_ipv4 = "[0-9]{3}(\\.[0-9]{1,3}){3}";    

            String ipv4addr1 = "192.168.1.102";
            String ipv4addr2 = "192.168";
            System.out.printf("%-40s = %b\n", "ipv4addr1.matches()", ipv4addr1.matches(reg_ipv4));
            System.out.printf("%-40s = %b\n", "ipv4addr2.matches()", ipv4addr2.matches(reg_ipv4));

            System.out.println();
        }
    }

运行结果：

    -------------------------------- testCompareAPIs ------------------------------
    str=abcAbcABCabCAbCabc
    str.equals("abcAbcABCabCAbCabc")                   = true
    str.equalsIgnoreCase("ABCABCABCABCABCABC")         = true
    str.compareTo("abce")                    = -36
    str.compareToIgnoreCase("ABC")           = 15
    str.startsWith("ab")                     = true
    str.startsWith("Ab")                     = true
    str.endsWith("bc")                       = true
    str.contains("ABC")                      = true
    regionMatches(cAbcABCabCAbCabc) = true
    regionMatches(CABCABCABCABCABC) = false
    regionMatches(cabcabcabcabcabc) = false
    str.contentEquals(new StringBuffer("abcAbcABCabCAbCabc"))    = true
    str.contentEquals(new StringBuilder("abcAbcABCabCAbCabc"))   = true
    ipv4addr1.matches()                      = true
    ipv4addr2.matches()                      = false

 
## 3.6 String “修改(追加/替换/截取/分割)”操作的API

源码如下(StringModifyTest.java):

    /**
     * String 中 修改(追加/替换/截取/分割)字符串的相关API演示
     *
     * @author skywang
     */

    public class StringModifyTest {
        
        public static void main(String[] args) {
            testModifyAPIs() ;
        }

        /**
         * String 中 修改(追加/替换/截取/分割)字符串的相关API演示
         */
        private static void testModifyAPIs() {
            System.out.println("-------------------------------- testModifyAPIs -------------------------------");

            String str = " abcAbcABCabCAbCabc ";
            System.out.printf("str=%s, len=%d\n", str, str.length());

            // 1.追加
            // 将"123"追加到str之后
            System.out.printf("%-30s = %s\n", "str.concat(\"123\")", 
                    str.concat("123"));

            // 2.截取
            // 截取str中从位置7(包括)开始的元素。
            System.out.printf("%-30s = %s\n", "str.substring(7)", str.substring(7));
            // 截取str中从位置7(包括)到位置10(不包括)之间的元素。
            System.out.printf("%-30s = %s\n", "str.substring(7, 10)", str.substring(7, 10));
            // 删除str中首位的空格，并返回。
            System.out.printf("%-30s = %s, len=%d\n", "str.trim()", str.trim(), str.trim().length());

            // 3.替换
            // 将str中的 “字符‘a’” 全部替换为 “字符‘_’”
            System.out.printf("%-30s = %s\n", "str.replace('a', 'M')", str.replace('a', '_'));
            // 将str中的第一次出现的“字符串“a”” 替换为 “字符串“###””
            System.out.printf("%-30s = %s\n", "str.replaceFirst(\"a\", \"###\")", str.replaceFirst("a", "###"));
            // 将str中的 “字符串“a”” 全部替换为 “字符串“$$$””
            System.out.printf("%-30s = %s\n", "str.replace(\"a\", \"$$$\")", str.replace("a", "$$$"));

            // 4.分割
            // 以“b”作为分隔符，对str进行分割
            String[] splits = str.split("b");
            for (int i=0; i<splits.length; i++) {
                System.out.printf("splits[%d]=%s\n", i, splits[i]);
            }

            System.out.println();
        }
    }

运行结果：

    -------------------------------- testModifyAPIs -------------------------------
    str= abcAbcABCabCAbCabc , len=20
    str.concat("123")              =  abcAbcABCabCAbCabc 123
    str.substring(7)               = ABCabCAbCabc 
    str.substring(7, 10)           = ABC
    str.trim()                     = abcAbcABCabCAbCabc, len=18
    str.replace('a', 'M')          =  _bcAbcABC_bCAbC_bc 
    str.replaceFirst("a", "###")   =  ###bcAbcABCabCAbCabc 
    str.replace("a", "$$$")        =  $$$bcAbcABC$$$bCAbC$$$bc 
    splits[0]= a
    splits[1]=cA
    splits[2]=cABCa
    splits[3]=CA
    splits[4]=Ca
    splits[5]=c 

 
## 3.7 String 操作Unicode的API

源码如下(StringUnicodeTest.java):

    /**
     * String 中与unicode相关的API
     *
     * @author skywang
     */

    public class StringUnicodeTest {
        
        public static void main(String[] args) {
            testUnicodeAPIs() ;
        }

        /**
         * String 中与unicode相关的API
         */
        private static void testUnicodeAPIs() {
            System.out.println("-------------------------------- testUnicodeAPIs ------------------------------");

            String ustr = new String(new int[] {0x5b57, 0x7b26, 0x7f16, 0x7801}, 0, 4);  // "字符编码"(\u5b57是‘字’的unicode编码)。0表示起始位置，4表示长度。
            System.out.printf("ustr=%s\n", ustr);

            //  获取位置0的元素对应的unciode编码
            System.out.printf("%-30s = 0x%x\n", "ustr.codePointAt(0)", ustr.codePointAt(0));

            // 获取位置2之前的元素对应的unciode编码
            System.out.printf("%-30s = 0x%x\n", "ustr.codePointBefore(2)", ustr.codePointBefore(2));

            // 获取位置1开始偏移2个代码点的索引
            System.out.printf("%-30s = %d\n", "ustr.offsetByCodePoints(1, 2)", ustr.offsetByCodePoints(1, 2));

            // 获取第0~3个元素之间的unciode编码的个数
            System.out.printf("%-30s = %d\n", "ustr.codePointCount(0, 3)", ustr.codePointCount(0, 3));

            System.out.println();
        }
    }

运行结果：

    -------------------------------- testUnicodeAPIs ------------------------------
    ustr=字符编码
    ustr.codePointAt(0)            = 0x5b57
    ustr.codePointBefore(2)        = 0x7b26
    ustr.offsetByCodePoints(1, 2)  = 3
    ustr.codePointCount(0, 3)      = 3

 
## 3.8 String 剩余的API

源码如下(StringOtherTest.java):

    /**
     * String 中其它的API
     *
     * @author skywang
     */

    public class StringOtherTest {
        
        public static void main(String[] args) {
            testOtherAPIs() ;
        }

        /**
         * String 中其它的API
         */
        private static void testOtherAPIs() {
            System.out.println("-------------------------------- testOtherAPIs --------------------------------");

            String str = "0123456789";
            System.out.printf("str=%s\n", str);

            // 1. 字符串长度
            System.out.printf("%s = %d\n", "str.length()", str.length());

            // 2. 字符串是否为空
            System.out.printf("%s = %b\n", "str.isEmpty()", str.isEmpty());

            // 3. [字节] 获取字符串对应的字节数组
            byte[] barr = str.getBytes();
            for (int i=0; i<barr.length; i++) {
                   System.out.printf("barr[%d]=0x%x ", i, barr[i]);
            }
            System.out.println();

            // 4. [字符] 获取字符串位置4的字符
            System.out.printf("%s = %c\n", "str.charAt(4)", str.charAt(4));

            // 5. [字符] 获取字符串对应的字符数组
            char[] carr = str.toCharArray();
            for (int i=0; i<carr.length; i++) {
                   System.out.printf("carr[%d]=%c ", i, carr[i]);
            }
            System.out.println();

            // 6. [字符] 获取字符串中部分元素对应的字符数组
            char[] carr2 = new char[3];
            str.getChars(6, 9, carr2, 0);
            for (int i=0; i<carr2.length; i++) {
                   System.out.printf("carr2[%d]=%c ", i, carr2[i]);
            }
            System.out.println();

            // 7. [字符] 获取字符数组对应的字符串
            System.out.printf("%s = %s\n", 
                    "str.copyValueOf(new char[]{'a','b','c','d','e'})", 
                    String.copyValueOf(new char[]{'a','b','c','d','e'}));

            // 8. [字符] 获取字符数组中部分元素对应的字符串
            System.out.printf("%s = %s\n", 
                    "str.copyValueOf(new char[]{'a','b','c','d','e'}, 1, 4)", 
                    String.copyValueOf(new char[]{'a','b','c','d','e'}, 1, 4));

            // 9. format()示例，将对象数组按指定格式转换为字符串
            System.out.printf("%s = %s\n", 
                    "str.format()", 
                    String.format("%s-%d-%b", "abc", 3, true));

            System.out.println();
        }
    }

运行结果：

    -------------------------------- testOtherAPIs --------------------------------
    str=0123456789
    str.length() = 10
    str.isEmpty() = false
    barr[0]=0x30 barr[1]=0x31 barr[2]=0x32 barr[3]=0x33 barr[4]=0x34 barr[5]=0x35 barr[6]=0x36 barr[7]=0x37 barr[8]=0x38 barr[9]=0x39 
    str.charAt(4) = 4
    carr[0]=0 carr[1]=1 carr[2]=2 carr[3]=3 carr[4]=4 carr[5]=5 carr[6]=6 carr[7]=7 carr[8]=8 carr[9]=9 
    carr2[0]=6 carr2[1]=7 carr2[2]=8 
    str.copyValueOf(new char[]{'a','b','c','d','e'}) = abcde
    str.copyValueOf(new char[]{'a','b','c','d','e'}, 1, 4) = bcde
    str.format() = abc-3-true

 
<a name="anchor4"></a>
# 4. String 完整示例

下面的示例是整合上面的几个示例的完整的String演示程序，源码如下(StringAPITest.java):

    /**
     * String 演示程序
     *
     * @author skywang
     */
    import java.util.HashMap;
    import java.nio.charset.Charset;
    import java.io.UnsupportedEncodingException;

    public class StringAPITest {
        
        public static void main(String[] args) {
            testStringConstructors() ; // String 构造函数测试程序
            testValueAPIs() ;          // String 的valueOf()演示程序
            testIndexAPIs() ;          // String 中index相关API演示
            testCompareAPIs() ;        // String 中比较相关API演示
            testModifyAPIs() ;         // String 中 修改(追加/替换/截取/分割)字符串的相关API演示
            testUnicodeAPIs() ;        // String 中与unicode相关的API
            testOtherAPIs() ;          // String 中其它的API
        }

        /**
         * String 构造函数测试程序
         */
        private static void testStringConstructors() {
            try {
                System.out.println("-------------------------------- testStringConstructors -----------------------");

                String str01 = new String();
                String str02 = new String("String02");
                String str03 = new String(new char[]{'s','t','r','0','3'});
                String str04 = new String(new char[]{'s','t','r','0','4'}, 1, 3);          // 1表示起始位置，3表示个数
                String str05 = new String(new byte[]{0x61, 0x62, 0x63, 0x64, 0x65});       // 0x61在ASC表中，对应字符"a"; 1表示起始位置，3表示长度
                String str06 = new String(new byte[]{0x61, 0x62, 0x63, 0x64, 0x65}, 1, 3); // 0x61在ASC表中，对应字符"a"; 1表示起始位置，3表示长度
                String str07 = new String(new byte[]{0x61, 0x62, 0x63, 0x64, 0x65}, 0);       // 0x61在ASC表中，对应字符"a";0，表示“高字节”
                String str08 = new String(new byte[]{0x61, 0x62, 0x63, 0x64, 0x65}, 0, 1, 3); // 0x61在ASC表中，对应字符"a"; 0，表示“高字节”；1表示起始位置，3表示长度
                String str09 = new String(new byte[]{(byte)0xe5, (byte)0xad, (byte)0x97, /* 字-对应的utf-8编码 */ 
                                                     (byte)0xe7, (byte)0xac, (byte)0xa6, /* 符-对应的utf-8编码 */ 
                                                     (byte)0xe7, (byte)0xbc, (byte)0x96, /* 编-对应的utf-8编码 */ 
                                                     (byte)0xe7, (byte)0xa0, (byte)0x81, /* 码-对应的utf-8编码 */ }, 
                                          0, 12, "utf-8");  // 0表示起始位置，12表示长度。
                String str10 = new String(new byte[]{(byte)0x5b, (byte)0x57, /* 字-对应的utf-16编码 */ 
                                                     (byte)0x7b, (byte)0x26, /* 符-对应的utf-16编码 */ 
                                                     (byte)0x7f, (byte)0x16, /* 编-对应的utf-16编码 */ 
                                                     (byte)0x78, (byte)0x01, /* 码-对应的utf-16编码 */ }, 
                                          0, 8, "utf-16");  // 0表示起始位置，8表示长度。
                String str11 = new String(new byte[]{(byte)0xd7, (byte)0xd6, /* 字-对应的gb2312编码  */ 
                                                     (byte)0xb7, (byte)0xfb, /* 符-对应的gb2312编码 */ 
                                                     (byte)0xb1, (byte)0xe0, /* 编-对应的gb2312编码 */ 
                                                     (byte)0xc2, (byte)0xeb, /* 码-对应的gb2312编码 */ }, 
                                          Charset.forName("gb2312")); 
                String str12 = new String(new byte[]{(byte)0xd7, (byte)0xd6, /* 字-对应的gbk编码 */ 
                                                     (byte)0xb7, (byte)0xfb, /* 符-对应的gbk编码 */ 
                                                     (byte)0xb1, (byte)0xe0, /* 编-对应的gbk编码 */ 
                                                     (byte)0xc2, (byte)0xeb, /* 码-对应的gbk编码 */ }, 
                                          0, 8, Charset.forName("gbk")); 
                String str13 = new String(new int[] {0x5b57, 0x7b26, 0x7f16, 0x7801}, 0, 4);  // "字符编码"(\u5b57是‘字’的unicode编码)。0表示起始位置，4表示长度。
                String str14 = new String(new StringBuffer("StringBuffer"));
                String str15 = new String(new StringBuilder("StringBuilder"));

                System.out.printf(" str01=%s \n str02=%s \n str03=%s \n str04=%s \n str05=%s \n str06=%s \n str07=%s \n str08=%s\n str09=%s\n str10=%s\n str11=%s\n str12=%s\n str13=%s\n str14=%s\n str15=%s\n",
                        str01, str02, str03, str04, str05, str06, str07, str08, str09, str10, str11, str12, str13, str14, str15);


                System.out.println();
            } catch (UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        }

        /**
         * String 中其它的API
         */
        private static void testOtherAPIs() {
            System.out.println("-------------------------------- testOtherAPIs --------------------------------");

            String str = "0123456789";
            System.out.printf("str=%s\n", str);

            // 1. 字符串长度
            System.out.printf("%s = %d\n", "str.length()", str.length());

            // 2. 字符串是否为空
            System.out.printf("%s = %b\n", "str.isEmpty()", str.isEmpty());

            // 3. [字节] 获取字符串对应的字节数组
            byte[] barr = str.getBytes();
            for (int i=0; i<barr.length; i++) {
                   System.out.printf("barr[%d]=0x%x ", i, barr[i]);
            }
            System.out.println();

            // 4. [字符] 获取字符串位置4的字符
            System.out.printf("%s = %c\n", "str.charAt(4)", str.charAt(4));

            // 5. [字符] 获取字符串对应的字符数组
            char[] carr = str.toCharArray();
            for (int i=0; i<carr.length; i++) {
                   System.out.printf("carr[%d]=%c ", i, carr[i]);
            }
            System.out.println();

            // 6. [字符] 获取字符串中部分元素对应的字符数组
            char[] carr2 = new char[3];
            str.getChars(6, 9, carr2, 0);
            for (int i=0; i<carr2.length; i++) {
                   System.out.printf("carr2[%d]=%c ", i, carr2[i]);
            }
            System.out.println();

            // 7. [字符] 获取字符数组对应的字符串
            System.out.printf("%s = %s\n", 
                    "str.copyValueOf(new char[]{'a','b','c','d','e'})", 
                    String.copyValueOf(new char[]{'a','b','c','d','e'}));

            // 8. [字符] 获取字符数组中部分元素对应的字符串
            System.out.printf("%s = %s\n", 
                    "str.copyValueOf(new char[]{'a','b','c','d','e'}, 1, 4)", 
                    String.copyValueOf(new char[]{'a','b','c','d','e'}, 1, 4));

            // 9. format()示例，将对象数组按指定格式转换为字符串
            System.out.printf("%s = %s\n", 
                    "str.format()", 
                    String.format("%s-%d-%b", "abc", 3, true));

            System.out.println();
        }

        /**
         * String 中 修改(追加/替换/截取/分割)字符串的相关API演示
         */
        private static void testModifyAPIs() {
            System.out.println("-------------------------------- testModifyAPIs -------------------------------");

            String str = " abcAbcABCabCAbCabc ";
            System.out.printf("%s, len=%d\n", str, str.length());

            // 1.追加
            // 将"123"追加到str之后
            System.out.printf("%-30s = %s\n", "str.concat(\"123\")", 
                    str.concat("123"));

            // 2.截取
            // 截取str中从位置7(包括)开始的元素。
            System.out.printf("%-30s = %s\n", "str.substring(7)", str.substring(7));
            // 截取str中从位置7(包括)到位置10(不包括)之间的元素。
            System.out.printf("%-30s = %s\n", "str.substring(7, 10)", str.substring(7, 10));
            // 删除str中首位的空格，并返回。
            System.out.printf("%-30s = %s, len=%d\n", "str.trim()", str.trim(), str.trim().length());

            // 3.替换
            // 将str中的 “字符‘a’” 全部替换为 “字符‘_’”
            System.out.printf("%-30s = %s\n", "str.replace('a', 'M')", str.replace('a', '_'));
            // 将str中的第一次出现的“字符串“a”” 替换为 “字符串“###””
            System.out.printf("%-30s = %s\n", "str.replaceFirst(\"a\", \"###\")", str.replaceFirst("a", "###"));
            // 将str中的 “字符串“a”” 全部替换为 “字符串“$$$””
            System.out.printf("%-30s = %s\n", "str.replace(\"a\", \"$$$\")", str.replace("a", "$$$"));

            // 4.分割
            // 以“b”作为分隔符，对str进行分割
            String[] splits = str.split("b");
            for (int i=0; i<splits.length; i++) {
                System.out.printf("splits[%d]=%s\n", i, splits[i]);
            }

            System.out.println();
        }


        /**
         * String 中比较相关API演示
         */
        private static void testCompareAPIs() {
            System.out.println("-------------------------------- testCompareAPIs ------------------------------");

            //String str = "abcdefghijklmnopqrstuvwxyz";
            String str = "abcAbcABCabCAbCabc";
            System.out.printf("%s\n", str);

            // 1. 比较“2个String是否相等”
            System.out.printf("%-50s = %b\n", 
                    "str.equals(\"abcAbcABCabCAbCabc\")", 
                    str.equals("abcAbcABCabCAbCabc"));

            // 2. 比较“2个String是否相等(忽略大小写)”
            System.out.printf("%-50s = %b\n", 
                    "str.equalsIgnoreCase(\"ABCABCABCABCABCABC\")", 
                    str.equalsIgnoreCase("ABCABCABCABCABCABC"));

            // 3. 比较“2个String的大小”
            System.out.printf("%-40s = %d\n", "str.compareTo(\"abce\")", str.compareTo("abce"));

            // 4. 比较“2个String的大小(忽略大小写)”
            System.out.printf("%-40s = %d\n", "str.compareToIgnoreCase(\"ABC\")", str.compareToIgnoreCase("ABC"));

            // 5. 字符串的开头是不是"ab"
            System.out.printf("%-40s = %b\n", "str.startsWith(\"ab\")", str.startsWith("ab"));

            // 6. 字符串的从位置3开头是不是"ab"
            System.out.printf("%-40s = %b\n", "str.startsWith(\"Ab\")", str.startsWith("Ab", 3));

            // 7. 字符串的结尾是不是"bc"
            System.out.printf("%-40s = %b\n", "str.endsWith(\"bc\")", str.endsWith("bc"));

            // 8. 字符串的是不是包含"ABC"
            System.out.printf("%-40s = %b\n", "str.contains(\"ABC\")", str.contains("ABC"));

            // 9. 比较2个字符串的部分内容
            String region1 = str.substring(2, str.length());    // 获取str位置3(包括)到末尾(不包括)的子字符串
            // 将“str中从位置2开始的字符串”和“region1中位置0开始的字符串”进行比较，比较长度是5。
            System.out.printf("regionMatches(%s) = %b\n", region1, 
                    str.regionMatches(2, region1, 0, 5));

            // 10. 比较2个字符串的部分内容(忽略大小写)
            String region2 = region1.toUpperCase();    // 将region1转换为大写
            String region3 = region1.toLowerCase();    // 将region1转换为小写
            System.out.printf("regionMatches(%s) = %b\n", region2, 
                    str.regionMatches(2, region2, 0, 5));
            System.out.printf("regionMatches(%s) = %b\n", region3, 
                    str.regionMatches(2, region3, 0, 5));

            // 11. 比较“String”和“StringBuffer”的内容是否相等
            System.out.printf("%-60s = %b\n", 
                    "str.contentEquals(new StringBuffer(\"abcAbcABCabCAbCabc\"))", 
                    str.contentEquals(new StringBuffer("abcAbcABCabCAbCabc")));

            // 12. 比较“String”和“StringBuilder”的内容是否相等
            System.out.printf("%-60s = %b\n", 
                    "str.contentEquals(new StringBuilder(\"abcAbcABCabCAbCabc\"))", 
                    str.contentEquals(new StringBuilder("abcAbcABCabCAbCabc")));

            // 13. match()测试程序
            // 正则表达式 xxx.xxx.xxx.xxx，其中xxx中x的取值可以是0～9，xxx中有1～3位。
            String reg_ipv4 = "[0-9]{3}(\\.[0-9]{1,3}){3}";    

            String ipv4addr1 = "192.168.1.102";
            String ipv4addr2 = "192.168";
            System.out.printf("%-40s = %b\n", "ipv4addr1.matches()", ipv4addr1.matches(reg_ipv4));
            System.out.printf("%-40s = %b\n", "ipv4addr2.matches()", ipv4addr2.matches(reg_ipv4));

            System.out.println();
        }

        /**
         * String 的valueOf()演示程序
         */
        private static void testValueAPIs() {
            System.out.println("-------------------------------- testValueAPIs --------------------------------");
            // 1. String    valueOf(Object obj)
            //  实际上，返回的是obj.toString();
            HashMap map = new HashMap();
            map.put("1", "one");
            map.put("2", "two");
            map.put("3", "three");
            System.out.printf("%-50s = %s\n", "String.valueOf(map)", String.valueOf(map));

            // 2.String    valueOf(boolean b)
            System.out.printf("%-50s = %s\n", "String.valueOf(true)", String.valueOf(true));

            // 3.String    valueOf(char c)
            System.out.printf("%-50s = %s\n", "String.valueOf('m')", String.valueOf('m'));

            // 4.String    valueOf(int i)
            System.out.printf("%-50s = %s\n", "String.valueOf(96)", String.valueOf(96));

            // 5.String    valueOf(long l)
            System.out.printf("%-50s = %s\n", "String.valueOf(12345L)", String.valueOf(12345L));

            // 6.String    valueOf(float f)
            System.out.printf("%-50s = %s\n", "String.valueOf(1.414f)", String.valueOf(1.414f));

            // 7.String    valueOf(double d)
            System.out.printf("%-50s = %s\n", "String.valueOf(3.14159d)", String.valueOf(3.14159d));

            // 8.String    valueOf(char[] data)
            System.out.printf("%-50s = %s\n", "String.valueOf(new char[]{'s','k','y'})", String.valueOf(new char[]{'s','k','y'}));

            // 9.String    valueOf(char[] data, int offset, int count)
            System.out.printf("%-50s = %s\n", "String.valueOf(new char[]{'s','k','y'}, 0, 2)", String.valueOf(new char[]{'s','k','y'}, 0, 2));

            System.out.println();
        }

        /**
         * String 中index相关API演示
         */
        private static void testIndexAPIs() {
            System.out.println("-------------------------------- testIndexAPIs --------------------------------");

            String istr = "abcAbcABCabCaBcAbCaBCabc";
            System.out.printf("istr=%s\n", istr);

            // 1. 从前往后，找出‘a’第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.indexOf((int)'a')", istr.indexOf((int)'a'));

            // 2. 从位置5开始，从前往后，找出‘a’第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.indexOf((int)'a', 5)", istr.indexOf((int)'a', 5));

            // 3. 从后往前，找出‘a’第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.lastIndexOf((int)'a')", istr.lastIndexOf((int)'a'));

            // 4. 从位置10开始，从后往前，找出‘a’第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.lastIndexOf((int)'a', 10)", istr.lastIndexOf((int)'a', 10));


            // 5. 从前往后，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.indexOf(\"bc\")", istr.indexOf("bc"));

            // 6. 从位置5开始，从前往后，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.indexOf(\"bc\", 5)", istr.indexOf("bc", 5));

            // 7. 从后往前，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.lastIndexOf(\"bc\")", istr.lastIndexOf("bc"));

            // 8. 从位置4开始，从后往前，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "istr.lastIndexOf(\"bc\", 4)", istr.lastIndexOf("bc", 4));

            System.out.println();
        }

        /**
         * String 中与unicode相关的API
         */
        private static void testUnicodeAPIs() {
            System.out.println("-------------------------------- testUnicodeAPIs ------------------------------");

            String ustr = new String(new int[] {0x5b57, 0x7b26, 0x7f16, 0x7801}, 0, 4);  // "字符编码"(\u5b57是‘字’的unicode编码)。0表示起始位置，4表示长度。
            System.out.printf("ustr=%s\n", ustr);

            //  获取位置0的元素对应的unciode编码
            System.out.printf("%-30s = 0x%x\n", "ustr.codePointAt(0)", ustr.codePointAt(0));

            // 获取位置2之前的元素对应的unciode编码
            System.out.printf("%-30s = 0x%x\n", "ustr.codePointBefore(2)", ustr.codePointBefore(2));

            // 获取位置1开始的元素对应的unciode编码
            System.out.printf("%-30s = %d\n", "ustr.offsetByCodePoints(1, 2)", ustr.offsetByCodePoints(1, 2));

            // 获取第0~3个元素之间的unciode编码的个数
            System.out.printf("%-30s = %d\n", "ustr.codePointCount(0, 3)", ustr.codePointCount(0, 3));

            System.out.println();
        }
    }
 
