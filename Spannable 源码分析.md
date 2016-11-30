<!-- MarkdownTOC depth=3 -->

- [1.简介](#1简介)
- [2.Spannable](#2spannable)
    - [2.1 SpannableString](#21-spannablestring)
    - [2.2 SpannableStringBuilder](#22-spannablestringbuilder)
- [3.CharacterStyle](#3characterstyle)
- [4.总结](#4总结)
- [5.参考](#5参考)

<!-- /MarkdownTOC -->


## 1.简介
[Spannable](https://developer.android.com/reference/android/text/Spannable.html) 是 Android 中用来给文字添加样式的接口，常见的子类的接口是 [Editable](https://developer.android.com/reference/android/text/Editable.html), [SpannableString](SpannableString), [SpannableStringBuilder](https://developer.android.com/reference/android/text/SpannableStringBuilder.html)，这篇文章主要分析下 SpannableString, SpannableStringBuilder 以及他们是如何在 Textview 中起作用的。


## 2.Spannable
Spannable 是一个接口，其中声明了如下两个方法, 下面主要介绍 setSpan 这个方法。

```java
public void removeSpan(Object what);
public void setSpan(Object what, int start, int end, int flags);
```

### 2.1 SpannableString
[SpannableString](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/java/android/text/SpannableString.java) 是由一个内部 [SpannableStringInternal](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/java/android/text/SpannableStringInternal.java) 实现的, 所以我们就直接看 SpannableStringInternal 源码:

```java
/* package */ 
void setSpan(Object what, int start, int end, int flags) {
    int nstart = start;
    int nend = end;

    // 检查 start 和 end 是否合法
    checkRange("setSpan", start, end);

    // 对 SPAN_PARAGRAPH 做格式检查
    if ((flags & Spannable.SPAN_PARAGRAPH) == Spannable.SPAN_PARAGRAPH) {
        if (start != 0 && start != length()) {
            char c = charAt(start - 1);
            if (c != '\n')
                throw new RuntimeException(
                        "PARAGRAPH span must start at paragraph boundary" +
                        " (" + start + " follows " + c + ")");
        }
        if (end != 0 && end != length()) {
            char c = charAt(end - 1);
            if (c != '\n')
                throw new RuntimeException(
                        "PARAGRAPH span must end at paragraph boundary" +
                        " (" + end + " follows " + c + ")");
        }
    }
    // 检查是否已经存在 span
    int count = mSpanCount;
    Object[] spans = mSpans;
    int[] data = mSpanData;
    for (int i = 0; i < count; i++) {
        if (spans[i] == what) {
            int ostart = data[i * COLUMNS + START];
            int oend = data[i * COLUMNS + END];
            data[i * COLUMNS + START] = start;
            data[i * COLUMNS + END] = end;
            data[i * COLUMNS + FLAGS] = flags;
            sendSpanChanged(what, ostart, oend, nstart, nend);
            return;
        }
    }
    // 检查数组大小
    if (mSpanCount + 1 >= mSpans.length) {
        Object[] newtags = ArrayUtils.newUnpaddedObjectArray(
                GrowingArrayUtils.growSize(mSpanCount));
        int[] newdata = new int[newtags.length * 3];
        System.arraycopy(mSpans, 0, newtags, 0, mSpanCount);
        System.arraycopy(mSpanData, 0, newdata, 0, mSpanCount * 3);
        mSpans = newtags;
        mSpanData = newdata;
    }
    mSpans[mSpanCount] = what;
    mSpanData[mSpanCount * COLUMNS + START] = start;
    mSpanData[mSpanCount * COLUMNS + END] = end;
    mSpanData[mSpanCount * COLUMNS + FLAGS] = flags;
    mSpanCount++;
    if (this instanceof Spannable)
        sendSpanAdded(what, nstart, nend);
}
```

首先是通过 checkRange 方法检查 start 和 end 是否合法：

```java
private void checkRange(final String operation, int start, int end) {
    if (end < start) {
        throw new IndexOutOfBoundsException(operation + " " +
                                            region(start, end) +
                                            " has end before start");
    }
    int len = length();
    if (start > len || end > len) {
        throw new IndexOutOfBoundsException(operation + " " +
                                            region(start, end) +
                                            " ends beyond length " + len);
    }
    if (start < 0 || end < 0) {
        throw new IndexOutOfBoundsException(operation + " " +
                                            region(start, end) +
                                            " starts before 0");
    }
}
```

接下来检查是否是 SPAN_PARAGRAPH, 当 flag 是 SPAN_PARAGRAPH 时, start 和 end 必须要是字符串的开头和结尾，例如：

```java
SpannableString ss = new SpannableString("abcd");
ss.setSpan(new UnderlineSpan(), 0,4, Spanned.SPAN_PARAGRAPH);
```

然后会检查是否已经存在了相同的 span 对象, 如果已经存在相同的 span, 那么就会用新的覆盖旧的, 所以在使用的时候要注意不要使用同一个对象:

```java
SpannableString ss = new SpannableString("abcd");
UnderlineSpan underlineSpan = new UnderlineSpan();

ss.setSpan(underlineSpan, 1,2, Spanned.SPAN_INCLUSIVE_INCLUSIVE);  // 这个被覆盖了
ss.setSpan(underlineSpan, 3,4, Spanned.SPAN_INCLUSIVE_INCLUSIVE);
```

然后检查数组大小, 不够大时扩容, 最终将值保存起来, 保存的方式也很容易理解, 如下代码所示:
```java
private static final int START = 0;
private static final int END = 1;
private static final int FLAGS = 2;
private static final int COLUMNS = 3;

mSpans[mSpanCount] = what;
mSpanData[mSpanCount * COLUMNS + START] = start;
mSpanData[mSpanCount * COLUMNS + END] = end;
mSpanData[mSpanCount * COLUMNS + FLAGS] = flags;
mSpanCount++;
```

到这里为止, 我们的 span 就被保存起来了。

### 2.2 SpannableStringBuilder
[SpannableStringBuilder](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/java/android/text/SpannableStringBuilder.java) 与 SpannableString 类似与 String 和 StringBuilder 之间的关系。SpannableStringBuilder 加入了 append, replace, delete 等方法，作用自不用多说，下面我们还是看下他的 setSpan:

```java
public void setSpan(Object what, int start, int end, int flags) {
    setSpan(true, what, start, end, flags);
}

// Note: if send is false, then it is the caller's responsibility to restore
// invariants. If send is false and the span already exists, then this method
// will not change the index of any spans.
private void setSpan(boolean send, Object what, int start, int end, int flags) {
    // 检查 start 和 end 是否合法
    checkRange("setSpan", start, end);
    int flagsStart = (flags & START_MASK) >> START_SHIFT;
    if(isInvalidParagraphStart(start, flagsStart)) {
        throw new RuntimeException("PARAGRAPH span must start at paragraph boundary");
    }
    int flagsEnd = flags & END_MASK;
    if(isInvalidParagraphEnd(end, flagsEnd)) {
        throw new RuntimeException("PARAGRAPH span must end at paragraph boundary");
    }
    // 0-length Spanned.SPAN_EXCLUSIVE_EXCLUSIVE
    if (flagsStart == POINT && flagsEnd == MARK && start == end) {
        if (send) {
            Log.e(TAG, "SPAN_EXCLUSIVE_EXCLUSIVE spans cannot have a zero length");
        }
        // Silently ignore invalid spans when they are created from this class.
        // This avoids the duplication of the above test code before all the
        // calls to setSpan that are done in this class
        return;
    }

    // 到这里都和之前的 SpannableString 没有什么区别

    // gap 的判断
    int nstart = start;
    int nend = end;
    if (start > mGapStart) {
        start += mGapLength;
    } else if (start == mGapStart) {
        if (flagsStart == POINT || (flagsStart == PARAGRAPH && start == length()))
            start += mGapLength;
    }
    if (end > mGapStart) {
        end += mGapLength;
    } else if (end == mGapStart) {
        if (flagsEnd == POINT || (flagsEnd == PARAGRAPH && end == length()))
            end += mGapLength;
    }

    // 检查是否已经添加过
    if (mIndexOfSpan != null) {
        Integer index = mIndexOfSpan.get(what);
        if (index != null) {
            int i = index;
            int ostart = mSpanStarts[i];
            int oend = mSpanEnds[i];
            if (ostart > mGapStart)
                ostart -= mGapLength;
            if (oend > mGapStart)
                oend -= mGapLength;
            mSpanStarts[i] = start;
            mSpanEnds[i] = end;
            mSpanFlags[i] = flags;
            if (send) {
                restoreInvariants();
                sendSpanChanged(what, ostart, oend, nstart, nend);
            }
            return;
        }
    }
    mSpans = GrowingArrayUtils.append(mSpans, mSpanCount, what);
    mSpanStarts = GrowingArrayUtils.append(mSpanStarts, mSpanCount, start);
    mSpanEnds = GrowingArrayUtils.append(mSpanEnds, mSpanCount, end);
    mSpanFlags = GrowingArrayUtils.append(mSpanFlags, mSpanCount, flags);
    mSpanOrder = GrowingArrayUtils.append(mSpanOrder, mSpanCount, mSpanInsertCount);
    invalidateIndex(mSpanCount);
    mSpanCount++;
    mSpanInsertCount++;
    // Make sure there is enough room for empty interior nodes.
    // This magic formula computes the size of the smallest perfect binary
    // tree no smaller than mSpanCount.
    int sizeOfMax = 2 * treeRoot() + 1;
    if (mSpanMax.length < sizeOfMax) {
        mSpanMax = new int[sizeOfMax];
    }
    if (send) {
        restoreInvariants();
        sendSpanAdded(what, nstart, nend);
    }
}
```

开始的操作依然是检查 start end 越界和 paragraph。接下来是对 gap 缓冲做的处理：
```java
int nstart = start;
int nend = end;
if (start > mGapStart) {
    start += mGapLength;
} else if (start == mGapStart) {
    if (flagsStart == POINT || (flagsStart == PARAGRAPH && start == length()))
        start += mGapLength;
}
if (end > mGapStart) {
    end += mGapLength;
} else if (end == mGapStart) {
    if (flagsEnd == POINT || (flagsEnd == PARAGRAPH && end == length()))
        end += mGapLength;
}
```

在这里会对 flags 有一个判断，当 flags 前后是 INCLUSIVE 且给的 start 和 end 是开头或结尾的时候，当这 append 的时候结尾或开始的字符串就会自动应用到这个 span。例如下面的代码：

```java
SpannableStringBuilder ssb = new SpannableStringBuilder("abcd");
ssb.setSpan(new UnderlineSpan(), 1, 4, Spanned.SPAN_POINT_POINT);
ssb.append("aaaa");
ssb.append("aaaa");
```
在 setSpan 的时候是对 "bcd" 部分加了下划线，当后面两次连续 append 之后，新加入的 "aaaaaaaa" 也被自动加入了下划线。

这里对 gap 做一个解释，在 SpannableStringBuilder 创建的时候会建一个名为 mText 的 char[]，数组的 size 经过两个方法的处理就会得到一个合适大小的数组，数组在赋值后剩余出来的空间就是 gap buffer。大小合适的数组可以避免在 append 的过程中对数组频繁的 copy 操作。

```java
public SpannableStringBuilder(CharSequence text, int start, int end) {
    int srclen = end - start;
    
    if (srclen < 0) throw new StringIndexOutOfBoundsException();
    
    mText = ArrayUtils.newUnpaddedCharArray(GrowingArrayUtils.growSize(srclen));
    mGapStart = srclen;
    mGapLength = mText.length - srclen;
    ...
}

// GrowingArrayUtils
/**
 * Given the current size of an array, returns an ideal size to which the array should grow.
 * This is typically double the given size, but should not be relied upon to do so in the
 * future.
 */
public static int growSize(int currentSize) {
    return currentSize <= 4 ? 8 : currentSize * 2;
}

// ArrayUtils
public static char[] newUnpaddedCharArray(int minLen) {
    return (char[])VMRuntime.getRuntime().newUnpaddedArray(char.class, minLen);
}

/**
 * Returns an array allocated in an area of the Java heap where it will never be moved.
 * This is used to implement native allocations on the Java heap, such as DirectByteBuffers
 * and Bitmaps.
 */
public native Object newNonMovableArray(Class<?> componentType, int length);
```

然后就是对 span 的检查，这一点的原理和 SpannableString 类似，也是重复使用的话只会保留最后设置的效果。下面操作就是对新加入的 span 的处理，同样使用构造函数中类似的对数组扩充的方式：

```java
mSpans = GrowingArrayUtils.append(mSpans, mSpanCount, what);
mSpanStarts = GrowingArrayUtils.append(mSpanStarts, mSpanCount, start);
mSpanEnds = GrowingArrayUtils.append(mSpanEnds, mSpanCount, end);
mSpanFlags = GrowingArrayUtils.append(mSpanFlags, mSpanCount, flags);
mSpanOrder = GrowingArrayUtils.append(mSpanOrder, mSpanCount, mSpanInsertCount);
// 更新 mIndexOfSpan 
invalidateIndex(mSpanCount);
mSpanCount++;
mSpanInsertCount++;

// Call this on any update to mSpans[], so that mIndexOfSpan can be updated
private void invalidateIndex(int i) {
    mLowWaterMark = Math.min(i, mLowWaterMark);
}

// GrowingArrayUtils 
public static <T> T[] append(T[] array, int currentSize, T element) {
    assert currentSize <= array.length;
    if (currentSize + 1 > array.length) {
        @SuppressWarnings("unchecked")
        T[] newArray = ArrayUtils.newUnpaddedArray(
                (Class<T>) array.getClass().getComponentType(), growSize(currentSize));
        System.arraycopy(array, 0, newArray, 0, currentSize);
        array = newArray;
    }
    array[currentSize] = element;
    return array;
}
```

这些 span 被存在一个线性的数组中，数组的排序是按照 start 的值建立的满二叉树来实现的，做成这样也是为了查询的更快。在这里的公式 2 * treeRoot() + 1 可以算出当前树的最大节点数，例如根节点的下标是 3，那么这个满二叉树中节点个数就是 7。完整的二叉树定义以及如何使用可以看 treeRoot 和 calcMax 方法的注释：

```java
// Make sure there is enough room for empty interior nodes.
// This magic formula computes the size of the smallest perfect binary
// tree no smaller than mSpanCount.
int sizeOfMax = 2 * treeRoot() + 1;
if (mSpanMax.length < sizeOfMax) {
    mSpanMax = new int[sizeOfMax];
}
if (send) {
    restoreInvariants();  // 对二叉树进行
    sendSpanAdded(what, nstart, nend);
}

// The spans (along with start and end offsets and flags) are stored in linear arrays sorted
// by start offset. For fast searching, there is a binary search structure imposed over these
// arrays. This structure is inorder traversal of a perfect binary tree, a slightly unusual
// but advantageous approach.

// The value-containing nodes are indexed 0 <= i < n (where n = mSpanCount), thus preserving
// logic that accesses the values as a contiguous array. Other balanced binary tree approaches
// (such as a complete binary tree) would require some shuffling of node indices.

// Basic properties of this structure: For a perfect binary tree of height m:
// The tree has 2^(m+1) - 1 total nodes.
// The root of the tree has index 2^m - 1.
// All leaf nodes have even index, all interior nodes odd.
// The height of a node of index i is the number of trailing ones in i's binary representation.
// The left child of a node i of height h is i - 2^(h - 1).
// The right child of a node i of height h is i + 2^(h - 1).

// Note that for arbitrary n, interior nodes of this tree may be >= n. Thus, the general
// structure of a recursive traversal of node i is:
// * traverse left child if i is an interior node
// * process i if i < n
// * traverse right child if i is an interior node and i < n
private int treeRoot() {
    return Integer.highestOneBit(mSpanCount) - 1;
}

// The span arrays are also augmented by an mSpanMax[] array that represents an interval tree
// over the binary tree structure described above. For each node, the mSpanMax[] array contains
// the maximum value of mSpanEnds of that node and its descendants. Thus, traversals can
// easily reject subtrees that contain no spans overlapping the area of interest.
// Note that mSpanMax[] also has a valid valuefor interior nodes of index >= n, but which have
// descendants of index < n. In these cases, it simply represents the maximum span end of its
// descendants. This is a consequence of the perfect binary tree structure.
private int calcMax(int i) {
    ...
}
```

## 3.CharacterStyle
Span 有很多种，这里拿最简单的 CharacterStyle 来举例说明我们设置的 Span 最终是如何影响到文字绘制的。前面的代码中出现的 UnderlineSpan 就是 CharacterStyle 的子类之一，可以在[官网](https://developer.android.com/reference/android/text/style/CharacterStyle.html)上看到其他的子类。CharacterStyle 有一个抽象方法是 updateDrawState，下面是 UnderlineSpan 的简易版：

```java
public class UnderlineSpan extends CharacterStyle 
    implements UpdateAppearance, ParcelableSpan {
    ...

    @Override
    public void updateDrawState(TextPaint ds) {
        ds.setUnderlineText(true);
    }
}
```
从这个实现来看，在文字绘制的时候会将绘制文字的 TextPaint 传进来，在这里更新其设置后就能实现对应的文字效果。从 [TextView源码分析](https://github.com/7heaven/AndroidSdkSourceAnalysis/blob/master/article/textview%E6%BA%90%E7%A2%BC%E8%A7%A3%E6%9E%90.md) 中可以看出在对 Span 的处理上分为两步：

```java
TextLine tl = TextLine.obtain();

tl.set(paint, buf, start, end, dir, directions, hasTabOrEmoji, tabStops);
tl.draw(canvas, x, ltop, lbaseline, lbottom);
```
通过 TextLine.obtain() 从 shared pool 中获取一个 TextLine（这么做的原因是这个对象本身比较大而且使用次数多，为了避免多次创建导致的频繁 GC），然后通过 TextLine.set 对 TextLine 进行初始化，最后 draw 方法绘制出文字。从源码可以看出在 set 的时候保存了 Span：

```java
void set(TextPaint paint, CharSequence text, int start, int limit, int dir,
            Directions directions, boolean hasTabs, TabStops tabStops) {
    mPaint = paint;
    mText = text;
    mStart = start;
    mLen = limit - start;
    mDir = dir;
    mDirections = directions;
    if (mDirections == null) {
        throw new IllegalArgumentException("Directions cannot be null");
    }
    mHasTabs = hasTabs;
    mSpanned = null;
    boolean hasReplacement = false;
    if (text instanceof Spanned) {
        mSpanned = (Spanned) text;   // 保存 span
        mReplacementSpanSpanSet.init(mSpanned, start, limit);
        hasReplacement = mReplacementSpanSpanSet.numberOfSpans > 0;
    }
    ...
}
```

然后是 draw 方法：

```java
void draw(Canvas c, float x, int top, int y, int bottom) {
    if (!mHasTabs) {
        if (mDirections == Layout.DIRS_ALL_LEFT_TO_RIGHT) {
            drawRun(c, 0, mLen, false, x, top, y, bottom, false);
            return;
        }
        if (mDirections == Layout.DIRS_ALL_RIGHT_TO_LEFT) {
            drawRun(c, 0, mLen, true, x, top, y, bottom, false);
            return;
        }
    }
    ...
}

 private float drawRun(Canvas c, int start,
        int limit, boolean runIsRtl, float x, int top, int y, int bottom,
        boolean needWidth) {
    if ((mDir == Layout.DIR_LEFT_TO_RIGHT) == runIsRtl) {
        float w = -measureRun(start, limit, limit, runIsRtl, null);
        handleRun(start, limit, limit, runIsRtl, c, x + w, top,
                y, bottom, null, false);
        return w;
    }
    return handleRun(start, limit, limit, runIsRtl, c, x, top,
            y, bottom, null, needWidth);
}

private float handleRun(int start, int measureLimit,
            int limit, boolean runIsRtl, Canvas c, float x, int top, int y,
            int bottom, FontMetricsInt fmi, boolean needWidth) {
    ...
    // 解析 Span
    mCharacterStyleSpanSet.init(mSpanned, mStart + start, mStart + limit);
    ...
    for (int j = i, jnext; j < mlimit; j = jnext) {
        jnext = mCharacterStyleSpanSet.getNextTransition(mStart + j, mStart + inext) -
                mStart;
        int offset = Math.min(jnext, mlimit);
        wp.set(mPaint);
        // 遍历 Span
        for (int k = 0; k < mCharacterStyleSpanSet.numberOfSpans; k++) {
            // Intentionally using >= and <= as explained above
            if ((mCharacterStyleSpanSet.spanStarts[k] >= mStart + offset) ||
                    (mCharacterStyleSpanSet.spanEnds[k] <= mStart + j)) continue;
            CharacterStyle span = mCharacterStyleSpanSet.spans[k];
            span.updateDrawState(wp);  // updateDrawState!!!
        }
        // Only draw hyphen on last run in line
        if (jnext < mLen) {
            wp.setHyphenEdit(0);
        }
        x += handleText(wp, j, jnext, i, inext, runIsRtl, c, x,
                top, y, bottom, fmi, needWidth || jnext < measureLimit, offset);
    }
    ...
}
```
几经周折到了 handleRun 这里，首先是 CharacterStyleSpanSet 的初始化，初始化的时候会将 mSpanned 中所带的 Span 都解析出来具体的过程可以看 [SpanSet](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/text/SpanSet.java) 的源码，然后使用 updateDrawState 更新 TextPaint 加入想要的效果，最后通过 handleText 绘制。



## 4.总结
Span 存储及调用的基本过程就是这样，更详细的 Span 使用可以看参考中的《Spans，一个强大的概念》这个文章或其原文《Spans, a Powerful Concept》。



## 5.参考
[TextView预渲染研究](http://ragnraok.github.io/textview-pre-render-research.html)

[TextView源碼解析](https://github.com/7heaven/AndroidSdkSourceAnalysis/blob/master/article/textview%E6%BA%90%E7%A2%BC%E8%A7%A3%E6%9E%90.md)

[Spans，一个强大的概念](http://rocko.xyz/2015/03/04/%E3%80%90%E8%AF%91%E3%80%91Spans%EF%BC%8C%E4%B8%80%E4%B8%AA%E5%BC%BA%E5%A4%A7%E7%9A%84%E6%A6%82%E5%BF%B5/)

[Spans, a Powerful Concept.](http://flavienlaurent.com/blog/2014/01/31/spans/)

[Gap buffer](https://en.wikipedia.org/wiki/Gap_buffer)

[二叉树](https://zh.wikipedia.org/wiki/%E4%BA%8C%E5%8F%89%E6%A0%91)