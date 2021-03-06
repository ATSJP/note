[TOC]



## Pattern

### 贪婪与非贪婪匹配

#### 定义

- 贪婪匹配是尽可能匹配多的字符

- 非贪婪匹配就是尽叮能匹配少的字符

贪婪匹配的写法是`.*`，非贪婪匹配的写法是`.*?`，多了一个`?`

#### 举例

例如：

贪婪匹配

```java
/**
 * 贪婪匹配
 * 
 * 运行结果：
 * group:A12
 * group:3
 * replaceAll:
 */
@Test
public void test2() {
    Pattern pattern = Pattern.compile("WORD_(.*)(\\d+)_321");
    Matcher matcher = pattern.matcher("WORD_A123_321");
    while (matcher.find()) {
        System.out.println("group:" + matcher.group(1));
        System.out.println("group:" + matcher.group(2));
    }
    System.out.println("replaceAll:" + matcher.replaceAll(""));
}
```

非贪婪匹配

```java
/**
 * 非贪婪匹配
 * 
 * 运行结果：
 * group:A
 * group:123
 * replaceAll:
 */
@Test
public void test3() {
    Pattern pattern = Pattern.compile("WORD_(.*?)(\\d+)_321");
    Matcher matcher = pattern.matcher("WORD_A123_321");
    while (matcher.find()) {
        System.out.println("group:" + matcher.group(1));
        System.out.println("group:" + matcher.group(2));
    }
    System.out.println("replaceAll:" + matcher.replaceAll(""));
}
```

 但是这里需要注意，如果匹配的结果在字符结尾，.*?就有可能匹配不到任何结果了，因为他会尽可能匹配少的字符（换句话说，都到末尾了，我尽可能少，那就是不匹配），例：

```java
/**
 * 贪婪匹配
 * 
 * 运行结果：
 * group:A12
 * group:3
 * group:1
 * replaceAll:
 */
@Test
public void test4() {
    Pattern pattern = Pattern.compile("WORD_(.*)(\\d+)_32(.*)");
    Matcher matcher = pattern.matcher("WORD_A123_321");
    while (matcher.find()) {
        System.out.println("group:" + matcher.group(1));
        System.out.println("group:" + matcher.group(2));
        System.out.println("group:" + matcher.group(3));
    }
    System.out.println("replaceAll:" + matcher.replaceAll(""));
}

/**
 * 非贪婪匹配
 * 
 * 运行结果：
 * group:A
 * group:123
 * group:
 * replaceAll:1
 */
@Test
public void test5() {
    Pattern pattern = Pattern.compile("WORD_(.*?)(\\d+)_32(.*?)");
    Matcher matcher = pattern.matcher("WORD_A123_321");
    while (matcher.find()) {
        System.out.println("group:" + matcher.group(1));
        System.out.println("group:" + matcher.group(2));
        System.out.println("group:" + matcher.group(3));
    }
    System.out.println("replaceAll:" + matcher.replaceAll(""));
}
```

###  从Compile到Matcher

下面是使用正则的一种经典写法：

```java
    @Test
    public void test1() {
        Pattern pattern = Pattern.compile("\\d+");
        Matcher matcher = pattern.matcher("123A4234A234");
        while (matcher.find()) {
            System.out.println(matcher.group());
        }
    }
```

从这个写法中，我们不难看出其中两大重要方法：compile()、matcher()。

接下来我们将从这两个方法入手，深入看下，源码中如何玩转的。

#### Compile

阿里巴巴开发规范中，有这么一条：

> 【强制】在使用正则表达式时，利用好其预编译功能，可以有效加快正则匹配速度。
>
>   说明：不要在方法体内定义： Pattern pattern =  Pattern.compile(“ 规则 ”);

大概我们能猜到compile()会做一些耗时的事情，那么，带着我们的猜想去看看compile()干了啥。

```java
public static Pattern compile(String regex) {
    return new Pattern(regex, 0);
}
```

这边是主动调用了私有的一个构造器，我们接着看：

```java
    private Pattern(String p, int f) {
        pattern = p;
        flags = f;

        // to use UNICODE_CASE if UNICODE_CHARACTER_CLASS present
        if ((flags & UNICODE_CHARACTER_CLASS) != 0)
            flags |= UNICODE_CASE;

        // Reset group index count
        capturingGroupCount = 1;
        localCount = 0;

        if (pattern.length() > 0) {
            // 长度大于0调用compile()
            compile();
        } else {
            root = new Start(lastAccept);
            matchRoot = lastAccept;
        }
    }
```

仔细看下构造器，其他的东西都不怎么引人注目，也不会很耗时，只有compile()这个方法（切记此compile()，不要和Pattern.compile()混淆哈），可能有点神秘，我们继续往下看：

```java
/**
 * Copies regular expression to an int array and invokes the parsing
 * of the expression which will create the object tree.
 */
private void compile() {
    // Handle canonical equivalences
    if (has(CANON_EQ) && !has(LITERAL)) {
        normalize();
    } else {
        normalizedPattern = pattern;
    }
    patternLength = normalizedPattern.length();

    // Copy pattern to int array for convenience
    // Use double zero to terminate pattern
    temp = new int[patternLength + 2];

    hasSupplementary = false;
    int c, count = 0;
    // Convert all chars into code points
    for (int x = 0; x < patternLength; x += Character.charCount(c)) {
        c = normalizedPattern.codePointAt(x);
        if (isSupplementary(c)) {
            hasSupplementary = true;
        }
        temp[count++] = c;
    }

    patternLength = count;   // patternLength now in code points

    if (! has(LITERAL))
        RemoveQEQuoting();

    // Allocate all temporary objects here.
    buffer = new int[32];
    groupNodes = new GroupHead[10];
    namedGroups = null;

    if (has(LITERAL)) {
        // Literal pattern handling
        matchRoot = newSlice(temp, patternLength, hasSupplementary);
        matchRoot.next = lastAccept;
    } else {
        // Start recursive descent parsing
        // 开始递归下降解析,
        matchRoot = expr(lastAccept);
        // Check extra pattern characters
        if (patternLength != cursor) {
            if (peek() == ')') {
                throw error("Unmatched closing ')'");
            } else {
                throw error("Unexpected internal error");
            }
        }
    }

    // Peephole optimization
    if (matchRoot instanceof Slice) {
        root = BnM.optimize(matchRoot);
        if (root == matchRoot) {
            root = hasSupplementary ? new StartS(matchRoot) : new Start(matchRoot);
        }
    } else if (matchRoot instanceof Begin || matchRoot instanceof First) {
        root = matchRoot;
    } else {
        root = hasSupplementary ? new StartS(matchRoot) : new Start(matchRoot);
    }

    // Release temporary storage
    temp = null;
    buffer = null;
    groupNodes = null;
    patternLength = 0;
    compiled = true;
}
```

`matchRoot = expr(lastAccept);`，这一行，看起来有一点说法：

```java
/**
 * The expression is parsed with branch nodes added for alternations.
 * This may be called recursively to parse sub expressions that may
 * contain alternations.
 */
private Node expr(Node end) {
    Node prev = null;
    Node firstTail = null;
    Branch branch = null;
    Node branchConn = null;

    for (;;) {
        Node node = sequence(end);
        Node nodeTail = root;      //double return
        if (prev == null) {
            prev = node;
            firstTail = nodeTail;
        } else {
            // Branch
            if (branchConn == null) {
                branchConn = new BranchConn();
                branchConn.next = end;
            }
            if (node == end) {
                // if the node returned from sequence() is "end"
                // we have an empty expr, set a null atom into
                // the branch to indicate to go "next" directly.
                node = null;
            } else {
                // the "tail.next" of each atom goes to branchConn
                nodeTail.next = branchConn;
            }
            if (prev == branch) {
                branch.add(node);
            } else {
                if (prev == end) {
                    prev = null;
                } else {
                    // replace the "end" with "branchConn" at its tail.next
                    // when put the "prev" into the branch as the first atom.
                    firstTail.next = branchConn;
                }
                prev = branch = new Branch(prev, node, branchConn);
            }
        }
        if (peek() != '|') {
            return prev;
        }
        next();
    }
}
```

`Node node = sequence(end);`，其他的，看起来都是在组装Node，只有这个是在获取Node，我们接着看：

```java
/**
 * Parsing of sequences between alternations.
 */
private Node sequence(Node end) {
    Node head = null;
    Node tail = null;
    Node node = null;
LOOP:
    for (;;) {
        int ch = peek();
        switch (ch) {
        case '(':
            // Because group handles its own closure,
            // we need to treat it differently
            node = group0();
            // Check for comment or flag group
            if (node == null)
                continue;
            if (head == null)
                head = node;
            else
                tail.next = node;
            // Double return: Tail was returned in root
            tail = root;
            continue;
        case '[':
            node = clazz(true);
            break;
        case '\\':
            ch = nextEscaped();
            if (ch == 'p' || ch == 'P') {
                boolean oneLetter = true;
                boolean comp = (ch == 'P');
                ch = next(); // Consume { if present
                if (ch != '{') {
                    unread();
                } else {
                    oneLetter = false;
                }
                node = family(oneLetter, comp);
            } else {
                unread();
                node = atom();
            }
            break;
        case '^':
            next();
            if (has(MULTILINE)) {
                if (has(UNIX_LINES))
                    node = new UnixCaret();
                else
                    node = new Caret();
            } else {
                node = new Begin();
            }
            break;
        case '$':
            next();
            if (has(UNIX_LINES))
                node = new UnixDollar(has(MULTILINE));
            else
                node = new Dollar(has(MULTILINE));
            break;
        case '.':
            next();
            if (has(DOTALL)) {
                node = new All();
            } else {
                if (has(UNIX_LINES))
                    node = new UnixDot();
                else {
                    node = new Dot();
                }
            }
            break;
        case '|':
        case ')':
            break LOOP;
        case ']': // Now interpreting dangling ] and } as literals
        case '}':
            node = atom();
            break;
        case '?':
        case '*':
        case '+':
            next();
            throw error("Dangling meta character '" + ((char)ch) + "'");
        case 0:
            if (cursor >= patternLength) {
                break LOOP;
            }
            // Fall through
        default:
            node = atom();
            break;
        }

        node = closure(node);

        if (head == null) {
            head = tail = node;
        } else {
            tail.next = node;
            tail = node;
        }
    }
    if (head == null) {
        return end;
    }
    tail.next = end;
    root = tail;      //double return
    return head;
}
```

看到这里，好像有点清晰了，我们在来梳理下，`Node node = sequence(end);`负责根据不同的字符，产生不同的Node，然后拼接到Node中去，而`matchRoot = expr(lastAccept);`看起来就更清晰了，根据输入的字符串通过`Node node = sequence(end);`生成不同的Node，然后在连接在一起，所以我们可以很自信的讲出，原来我们写的正则表达式，最后会在compile()方法中，被`matchRoot = expr(lastAccept);`解析成一种单链表结构。当然compile()方法还有一些其他判断处理，这些对于常规的表达式来说不是重点，就现在而言，你可以认为她是在处理一些特殊情况，后续我们在详细过一遍源码的细节。

防止大家还有点模糊，说这一大堆文字，谁看的懂呀，不画个图意思意思？OK，图来：

```flow
st=>start: Pattern.compile()
o1=>operation: new Pattern(regex, 0)
o2=>operation: compile()
o3=>operation: matchRoot = expr(lastAccept)
o4=>operation: Node node = sequence(end);

c1=>condition: pattern.length()>0
c2=>condition: peek() != 竖线
end=>end: 放行

st->o1->c1
c1(yes)>o2->o3->o4->c2
c1(no)->end

c2(yes)->end
c2(no)->o4
```

这样够清晰了吗，嘻嘻，好叻这样的一个流程，我们应该十分清晰了，下面将从其他细节部分再次深入了解。

#### Matcher

好了，看完Compile，我们再来看看经典写法：

```java
    @Test
    public void test1() {
        Pattern pattern = Pattern.compile("\\d+");
        Matcher matcher = pattern.matcher("123A4234A234");
        while (matcher.find()) {
            System.out.println(matcher.group());
        }
    }
```

很明显，我们每次判断是否匹配上时，都通过`matcher.find()`进行查找。于是，我们大胆猜测，匹配的算法就在find方法中，让我们一起打开她的神秘面纱：

```java
public boolean find() {
    int nextSearchIndex = last;
    if (nextSearchIndex == first)
        nextSearchIndex++;

    // If next search starts before region, start it at region
    if (nextSearchIndex < from)
        nextSearchIndex = from;

    // If next search starts beyond region then it fails
    if (nextSearchIndex > to) {
        for (int i = 0; i < groups.length; i++)
            groups[i] = -1;
        return false;
    }
    return search(nextSearchIndex);
}
```

同样的，我们抛去不太重要的代码行，直接来看核心`search(nextSearchIndex)`：

```java
boolean search(int from) {
    this.hitEnd = false;
    this.requireEnd = false;
    from        = from < 0 ? 0 : from;
    this.first  = from;
    this.oldLast = oldLast < 0 ? from : oldLast;
    for (int i = 0; i < groups.length; i++)
        groups[i] = -1;
    acceptMode = NOANCHOR;
    boolean result = parentPattern.root.match(this, from, text);
    if (!result)
        this.first = -1;
    this.oldLast = this.last;
    return result;
}
```

可以非常清晰的看到` boolean result = parentPattern.root.match(this, from, text);`这一行，在结合我们上面Compile的探索，发现此行的目的为：从根节点Root开始match。

有心的小伙伴，肯定有些纳闷，单链表的访问至少也有个指针负责遍历吧，这里咋就看起来遍历一次，就没了呢，别急，继续往下看。

之前我们看Compile也讲过，Pattern类中有着不同的Node实现，她们之前有一个叫做Start的类，上面的parentPattern.root便是这个类的实例，下面我们来看看：

```java
    static class Node extends Object {
        Node next;
        Node() {
            next = Pattern.accept;
        }
        /**
         * This method implements the classic accept node.
         */
        boolean match(Matcher matcher, int i, CharSequence seq) {
            matcher.last = i;
            matcher.groups[0] = matcher.first;
            matcher.groups[1] = matcher.last;
            return true;
        }
        /**
         * This method is good for all zero length assertions.
         */
        boolean study(TreeInfo info) {
            if (next != null) {
                return next.study(info);
            } else {
                return info.deterministic;
            }
        }
    } 

    static class Start extends Node {
        int minLength;
        Start(Node node) {
            this.next = node;
            TreeInfo info = new TreeInfo();
            next.study(info);
            minLength = info.minLength;
        }
        boolean match(Matcher matcher, int i, CharSequence seq) {
            if (i > matcher.to - minLength) {
                matcher.hitEnd = true;
                return false;
            }
            int guard = matcher.to - minLength;
            for (; i <= guard; i++) {
                if (next.match(matcher, i, seq)) {
                    matcher.first = i;
                    matcher.groups[0] = matcher.first;
                    matcher.groups[1] = matcher.last;
                    return true;
                }
            }
            matcher.hitEnd = true;
            return false;
        }
        boolean study(TreeInfo info) {
            next.study(info);
            info.maxValid = false;
            info.deterministic = false;
            return false;
        }
    }
```

看到这边，是不是有点熟悉的味道了，有循环，有下一个节点，但是又有点不同，还是没有找到我们最熟悉的指针，仔细观察方法的返回值，是个boolean类型，那看起来是每个实现类都完成自己的匹配，并返回匹配结果，然后不属于自己匹配的部分，交由Next去匹配，我们暂且称作链表匹配，带着我们猜测，再去看看其他Node的实现：

```java
static final class UnixCaret extends Node {
    boolean match(Matcher matcher, int i, CharSequence seq) {
        int startIndex = matcher.from;
        int endIndex = matcher.to;
        if (!matcher.anchoringBounds) {
            startIndex = 0;
            endIndex = matcher.getTextLength();
        }
        // Perl does not match ^ at end of input even after newline
        if (i == endIndex) {
            matcher.hitEnd = true;
            return false;
        }
        if (i > startIndex) {
            char ch = seq.charAt(i-1);
            if (ch != '\n') {
                return false;
            }
        }
        return next.match(matcher, i, seq);
    }
}
```

```java
static final class UnixDollar extends Node {
    boolean multiline;
    UnixDollar(boolean mul) {
        multiline = mul;
    }
    boolean match(Matcher matcher, int i, CharSequence seq) {
        int endIndex = (matcher.anchoringBounds) ?
            matcher.to : matcher.getTextLength();
        if (i < endIndex) {
            char ch = seq.charAt(i);
            if (ch == '\n') {
                // If not multiline, then only possible to
                // match at very end or one before end
                if (multiline == false && i != endIndex - 1)
                    return false;
                // If multiline return next.match without setting
                // matcher.hitEnd
                if (multiline)
                    return next.match(matcher, i, seq);
            } else {
                return false;
            }
        }
        // Matching because at the end or 1 before the end;
        // more input could change this so set hitEnd
        matcher.hitEnd = true;
        // If a $ matches because of end of input, then more input
        // could cause it to fail!
        matcher.requireEnd = true;
        return next.match(matcher, i, seq);
    }
    boolean study(TreeInfo info) {
        next.study(info);
        return info.deterministic;
    }
}
```

等等。如果你有耐心，大可以看完所有的Node实现，最后一定会发现，每个Node完成自己的任务后，调用Next的Match方法。

最后，我们还是画上一张图，来说明下核心调用过程(涉及到的Node，仅为了演示，不分先后顺序)：

```flow
st=>start: Start
o1=>operation: UnixCaret
o2=>operation: Dollar
o3=>operation: ...
end=>end: End

st->o1->o2->o3->end

```





























