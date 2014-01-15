# Getting XHTML blocks by convention in Hannon Hill’s Cascade Server

In order to reduce the amount of configuration necessary, we will find a blocks
based on conventions. First we identify and define the conventions we wish to
use.

Let’s take the following three pages:

* /swosu/academics/socsci/programs/history/index
* /swosu/academics/pharmacy/gen-info/history
* /swosu/academics/ace/business/programs/index

Looking at the first example, we would expect the history department’s page to
show a block related to the history department if it exists, and a social
sciences block if not. And if there is no social sciences block, then we hope
to find either a general academics or swosu block.

In the second example, we would expect to find a pharmacy block, but *not* a
history block. Since the occasion for wanting a block for a single page and
not a folder of pages is rare, we should just ignore the last component of the
path when trying to determine which block is the best match.

In the last example, let’s pretend we want a different block for pages in the
programs folder. Since many departments have a folder called “programs,” we
want to find a block that is specific to business.

This gives us the following requirements:

* Ignore the last component of the path
* Give higher priority to blocks with names that correspond to components
  later in the path
* Give higher priority to blocks with more specific names than less, i.e.
  it has more of the path components in the name

In the first example, we want to search for blocks with the following names
in increasing order of priority:

* swosu
* academics
* swosu-academics
* socsci
* academics-socsci
* swosu-academics-socsci
* programs
* socsci-programs
* academics-socsci-programs
* swosu-academics-socsci-programs
* history
* programs-history
* socsci-programs-history
* academics-socsci-programs-history
* swosu-academics-socsci-programs-history

In this case, we will probably end up using the block named “history.” Now to
look at the code.

```xslt
<?xml version="1.0" encoding="utf-8"?>
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform" extension-element-prefixes="exsl str" version="1.0" xmlns:exsl="http://exslt.org/common" xmlns:str="http://exslt.org/strings">
  <xsl:output encoding="utf-8" indent="yes" method="xml" version="1.0"/>

  . . .

</xsl:stylesheet>
```

The only thing to notice in the XML declaration is that we are using EXSLT for
some additional functionality, namely `node-set` and `tokenize`.

```xslt
<xsl:template match="/system-index-block">
  <xsl:variable name="index" select="."/>
  <xsl:variable name="tokens">
    <xsl:apply-templates select="calling-page/system-page/path"/>
  </xsl:variable>
  <xsl:variable name="matches">
    <xsl:for-each select="exsl:node-set($tokens)/token">
      <xsl:copy-of select="$index/system-block[name/text()=current()]/block-xhtml"/>
    </xsl:for-each>
  </xsl:variable>
  <xsl:copy-of select="exsl:node-set($matches)/block-xhtml[position()=last()]/node()"/>
</xsl:template>
```

First we capture the `system-index-block` in a variable because we will need to
reference it later. When we call `apply-templates` on
`calling-page/system-page/path`, we capture some XML that corresponds to the
list of block names we defined above in a variable called `tokens`. The XML
should look like this:

```xml
<tokens>
  <token>swosu</token>
  <token>academics</token>
  <token>swosu-academics</token>
  <token>socsci</token>
  <token>academics-socsci</token>
  <token>swosu-academics-socsci</token>
  <token>programs</token>
  <token>socsci-programs</token>
  <token>academics-socsci-programs</token>
  <token>swosu-academics-socsci-programs</token>
  <token>history</token>
  <token>programs-history</token>
  <token>socsci-programs-history</token>
  <token>academics-socsci-programs-history</token>
  <token>swosu-academics-socsci-programs-history</token>
</tokens>
```

Since we are working on an Index Block, we reduce the set set of blocks to
those matching the names in tokens. The `for-each` will go through each token,
find a matching System Block, and copy it into a variable called `matches`.
The resulting XML is too verbose to paste below, but an abbreviated form could
look like this:

```xml
<!-- This might correspond to a System Block named 'academics' -->
<block-xhtml>
  <!-- XHTML that is pulled from 'academics' -->
</block-xhtml>
<block-xhtml>
  <!-- XHTML that is pulled from 'socsci' -->
</block-xhtml>
<block-xhtml>
  <!-- XHTML that pulled from 'history' -->
</block-xhtml>
```

Finally, we reduce the set even further to contain only the last `block-xhtml`
node of the above set, so we make a copy of the XHTML from the history block
above.

Everything else in the file is to generate the names for `tokens` above.

```xslt
<xsl:template match="path">
  <xsl:variable name="tokens">
    <tokens><xsl:copy-of select="str:tokenize(., '/')[not(position()=last())]"/></tokens>
  </xsl:variable>
  <xsl:apply-templates mode="hyphenate" select="exsl:node-set($tokens)"/>
</xsl:template>
```

The above bit of code simply splits the path into its components on the slash,
removes the last element, and stores the results in a the variable `tokens`.
That is `/swosu/academics/socsci/programs/history/index` becomes

```xml
<tokens>
  <token>swosu</token>
  <token>academics</token>
  <token>socsci</token>
  <token>programs</token>
  <token>history</token>
</tokens>
```

We then pass the set of tokens to another template that does the hyphenation
and returns the complete set.

The following two templates are where the magic happens. These are both
recursive; that is they call themselves.

```xlst
<xsl:template match="tokens" mode="hyphenate">
  <xsl:if test="token">
    <xsl:variable name="tokens">
      <tokens><xsl:copy-of select="token[not(position()=last())]"/></tokens>
    </xsl:variable>
    <xsl:apply-templates mode="hyphenate" select="exsl:node-set($tokens)"/>
    <xsl:apply-templates mode="end-hyphenation" select="."/>
  </xsl:if>
</xsl:template>
```

The first pass through the above template we remove the last element and call
the same template again. So when we re-enter the template, the XML will look
like:

```xml
<tokens>
  <token>swosu</token>
  <token>academics</token>
  <token>socsci</token>
  <token>programs</token>
</tokens>
```

It will then remove the last element again and call the same template with

```xml
<!-- Third entry -->
<tokens>
  <token>swosu</token>
  <token>academics</token>
  <token>socsci</token>
</tokens>

<!-- Fourth entry -->
<tokens>
  <token>swosu</token>
  <token>academics</token>
</tokens>

<!-- Fifth entry -->
<tokens>
  <token>swosu</token>
</tokens>
```

Because we are calling this template recursively, the last set of tokens will
be returned ahead of the fourth set which is returned before the third set and
so on. This is how we get the less specific items in front of the more specific
items.

We are now to the point where we have called the template with only one token,
we have no more tokens to remove, and we move on to the last line where we call
the template with `mode="end-hyphenation"`. The first thing that will be passed
to this template is the XML commented with “Fifth entry” above.

```xslt
<xsl:template match="tokens" mode="end-hyphenation">
  <xsl:if test="count(token) &gt; 1">
    <xsl:variable name="tokens">
      <tokens><xsl:copy-of select="token[not(position()=1)]"/></tokens>
    </xsl:variable>
    <xsl:apply-templates mode="end-hyphenation" select="exsl:node-set($tokens)"/>
  </xsl:if>
  <token>
    <xsl:for-each select="token">
      <xsl:value-of select="."/>
      <xsl:if test="not(position()=last())">-</xsl:if>
    </xsl:for-each>
  </token>
</xsl:template>
```

Since there is nothing more do for this set, it will be returned as:

```xml
<token>swosu</token>
```

Now we have moved back up the stack by one level into the “hyphenation” mode
with the set commented “Fourth entry” and pass it into the “end-hyphenation”
mode.

Since there is more than one token, we remove the *first* element and
pass the remaining tokens into the “end-hyphenation” token. So we just called
the template with the following set:

```xml
<token>academics</token>
```

Again, we are down to one token so we return it, but now we have to do
something with the set from “Fourth entry”, so we combine the token with
hyphens and return:

```xml
<token>swosu-academics</token>
```

Now this is starting to look like what we want. If we take the set commented
“Third entry,” we start with 3 items, pop of the front one, call
`end-hyphenation`, pop off the first item to give us one token, and then start
hypenating. This pass gives us the next three results.

```xml
<token>socsci</token>
<token>academics-socsci</token>
<token>swosu-academics-socsci</token>
```

And we continue to move back up the stack, calling `end-hyphenation` with the
“Second entry” we get

```xml
<token>programs</token>
<token>socsci-programs</token>
<token>academics-socsci-programs</token>
<toekn>swosu-academics-socsci-programs</toekn>
```

And finally, we are back to the first set where we have the most specific
matches.

```xml
<token>history</token>
<token>programs-history</token>
<token>socsci-programs-history</token>
<token>academics-socsci-programs-history</token>
<toekn>swosu-academics-socsci-programs-history</toekn>
```

We now have the complete set of possible names that can be matched which we
used above to reduce the Index Block to the matching XHTML blocks, and finally
took the very last, that is the most specific, XHTML block for use in the page.