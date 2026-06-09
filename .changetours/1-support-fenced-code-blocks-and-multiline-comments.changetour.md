---
schemaVersion: 1
prNumber: 1
prOwner: hannahpotter
prRepo: plume
baseSha: 84d3ebdb5b3d5c8fe4d92fe9a4fca0494ffebcd6
headSha: 30034dc234df0142234914c2fbe7bdeac8d51ef6
---
# Support fenced code blocks and multiline comments

This pull request enhances the `EntryReader` class by adding support for multiline comments and fenced code blocks, and updates its constructors and tests accordingly. The changes introduce new parameters and logic to handle these features, while also deprecating older constructors in favor of the new, more flexible ones.

## Multiline comment and fenced code block support <!-- collapsed -->

The feature adds regex-based multiline comment delimiters and fenced code block tracking, allowing `EntryReader` to skip content inside multiline comments while preserving it inside fenced code blocks.

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · /** Regular expression that matches the start of a multilin…</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

```diff
@@ -90,6 +90,12 @@ public class EntryReader extends LineNumberReader implements Iterable<String>, I
    */
   private final @Nullable Pattern commentRegex;
 
+  /** Regular expression that matches the start of a multiline comment. */
+  private final @Nullable Pattern multilineCommentStart;
+
+  /** Regular expression that matches the end of a multiline comment. */
+  private final @Nullable Pattern multilineCommentEnd;
+
   /**
    * Regular expression that starts a long entry.
    *
```

</details>

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · /** True if currently inside a multiline comment &lt;!-- ... -…</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 baseBlob=7b6259b6a46aa13c61d4bc6fc8aa32cc87defdfb -->

```diff
@@ -129,13 +135,56 @@ public class EntryReader extends LineNumberReader implements Iterable<String>, I
   /** Platform-specific line separator. */
   private static final String lineSep = System.lineSeparator();
 
+  /** True if currently inside a multiline comment <!-- ... --> .*/
+  private boolean inMultilineComment = false;
+
+  /** True if currently inside a fenced code block (``` ... ```). */
+  private boolean inFencedCodeBlock = false;
+
   // ///////////////////////////////////////////////////////////////////////////
   // Constructors
   //
 
   // Inputstream and charset constructors
 
-  // This is the complete constructor that supplies all possible arguments.
+  /**
+   * Create an EntryReader.
+   *
+   * @param in source from which to read entries
+   * @param charsetName the character set to use
+   * @param filename non-null file name for stream being read
+   * @param twoBlankLines true if entries are separated by two blank lines rather than one
+   * @param commentRegexString regular expression that matches comments. Any text that matches
+   *     commentRegex is removed. A line that is entirely a comment is ignored.
+   * @param includeRegexString regular expression that matches include directives. The expression
+   *     should define one group that contains the include file name.
+   * @param multilineCommentStart regular expression that matches the start of a multiline comment.
+   *     Any text that matches and follows after this regex is removed.
+   * @param multilineCommentEnd regular expression that matches the end of a multiline comment. Any
+   *     text that matches and precedes this regex is removed.
+   * @throws UnsupportedEncodingException if the charset encoding is not supported
+   * @see #EntryReader(InputStream,String,String,String)
+   */
+  public @MustCallAlias EntryReader(
+      @MustCallAlias InputStream in,
+      String charsetName,
+      String filename,
+      boolean twoBlankLines,
+      @Nullable @Regex String commentRegexString,
+      @Nullable @Regex(1) String includeRegexString,
+      @Nullable @Regex String multilineCommentStart,
+      @Nullable @Regex String multilineCommentEnd)
+      throws UnsupportedEncodingException {
+    this(
+        new InputStreamReader(in, charsetName),
+        filename,
+        twoBlankLines,
+        commentRegexString,
+        includeRegexString,
+        multilineCommentStart,
+        multilineCommentEnd);
+  }
+
   /**
    * Create an EntryReader that uses the given character set.
    *
```

</details>
