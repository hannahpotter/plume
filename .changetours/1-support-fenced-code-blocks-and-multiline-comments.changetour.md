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
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · /** True if currently inside a multiline comment .*/</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

```diff
@@ -129,13 +135,56 @@ public class EntryReader extends LineNumberReader implements Iterable<String>, I
   /** Platform-specific line separator. */
   private static final String lineSep = System.lineSeparator();
 
+  /** True if currently inside a multiline comment .*/
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

## Reading logic

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · Declares fields to track whether the reader is inside a fenced code block and multiline comment</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 highlights=new:91-93,old:91-92 summary="Declares fields to track whether the reader is inside a fenced code block and multiline comment" baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

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

The `readEntry` method now tracks fenced code blocks (toggling state when it encounters ` ``` `) and processes multiline comment boundaries, skipping lines inside multiline comments while preserving everything inside fenced code blocks.

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · // Handles fenced code blocks.</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 highlights=new:857 baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

```diff
@@ -656,6 +853,39 @@ public void setDebug(boolean debug) {
     }
 
     String line = getNextLine();
+    // Handles fenced code blocks.
+    if (line != null && line.trim().startsWith("```")) {
+      inFencedCodeBlock = !inFencedCodeBlock;
+      return line;
+    }
+    if (inFencedCodeBlock) {
+      return line;
+    }
+
+    // Handles multiline comments.
+    // Multiline comments are block-level only: a block must start with
+    // multilineCommentStart on its own line and end with
+    // multilineCommentEnd on its own line.
+    // All lines inside a multiline comment are ignored.
+
+    if (line == null) {
+      return null;
+    }
+
+    String trimmed = line.trim();
+
+    if (inMultilineComment) {
+      if (multilineCommentEnd != null && multilineCommentEnd.matcher(trimmed).matches()) {
+        inMultilineComment = false;
+      }
+      return "";
+    }
+
+    if (multilineCommentStart != null && multilineCommentStart.matcher(trimmed).matches()) {
+      inMultilineComment = true;
+      return "";
+    }
+
     if (commentRegex != null) {
       while (line != null) {
         Matcher cmatch = commentRegex.matcher(line);
```

</details>

## Constructor updates

All constructors now accept `multilineCommentStart` and `multilineCommentEnd` parameters. The primary InputStream constructor documents the new parameters and shows how the patterns are compiled and stored. A clarifying comment notes that all other constructors delegate to this one via `this(...)`.

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · // The body of every constructor other than this one is an…</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

```diff
@@ -280,6 +332,7 @@ public class EntryReader extends LineNumberReader implements Iterable<String>, I
     this(in, "(InputStream)", null, null);
   }
 
+  // The body of every constructor other than this one is an invocation of `this(...)`.
   /**
    * Create an EntryReader.
    *
```

</details>

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · * @param multilineCommentStart regular expression that matc…</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

```diff
@@ -290,14 +343,20 @@ public class EntryReader extends LineNumberReader implements Iterable<String>, I
    *     commentRegex is removed. A line that is entirely a comment is ignored
    * @param includeRegexString regular expression that matches include directives. The expression
    *     should define one group that contains the include file name
+   * @param multilineCommentStart regular expression that matches the start of a multiline comment.
+   *     Any text that matches and follows after this regex is removed.
+   * @param multilineCommentEnd regular expression that matches the end of a multiline comment. Any
+   *     text that matches and precedes this regex is removed.
    */
   @SuppressWarnings("builder") // storing into a collection
   public @MustCallAlias EntryReader(
       @MustCallAlias Reader reader,
       String filename,
       boolean twoBlankLines,
       @Nullable @Regex String commentRegexString,
-      @Nullable @Regex(1) String includeRegexString) {
+      @Nullable @Regex(1) String includeRegexString,
+      @Nullable @Regex String multilineCommentStart,
+      @Nullable @Regex String multilineCommentEnd) {
     // We won't use superclass methods, but passing null as an argument
     // leads to a NullPointerException.
     super(DummyReader.it);
```

</details>

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · if (multilineCommentStart == null) {</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

```diff
@@ -313,6 +372,39 @@ public class EntryReader extends LineNumberReader implements Iterable<String>, I
     } else {
       includeRegex = Pattern.compile(includeRegexString);
     }
+    if (multilineCommentStart == null) {
+      this.multilineCommentStart = null;
+    } else {
+      this.multilineCommentStart = Pattern.compile(multilineCommentStart);
+    }
+    if (multilineCommentEnd == null) {
+      this.multilineCommentEnd = null;
+    } else {
+      this.multilineCommentEnd = Pattern.compile(multilineCommentEnd);
+    }
+  }
+
+  /**
+   * Create an EntryReader.
+   *
+   * @param reader source from which to read entries
+   * @param filename file name corresponding to reader, for use in error messages
+   * @param twoBlankLines true if entries are separated by two blank lines rather than one
+   * @param commentRegexString regular expression that matches comments. Any text that matches
+   *     commentRegex is removed. A line that is entirely a comment is ignored
+   * @param includeRegexString regular expression that matches include directives. The expression
+   *     should define one group that contains the include file name
+   * @deprecated use {@link #EntryReader(Reader,String,boolean,String,String,String,String)}
+   */
+  @Deprecated // 2026-01-05
+  @SuppressWarnings("builder") // storing into a collection
+  public @MustCallAlias EntryReader(
+      @MustCallAlias Reader reader,
+      String filename,
+      boolean twoBlankLines,
+      @Nullable @Regex String commentRegexString,
+      @Nullable @Regex(1) String includeRegexString) {
+    this(reader, filename, twoBlankLines, commentRegexString, includeRegexString, null, null);
   }
 
   /**
```

</details>

The existing constructors without the new parameters are deprecated in favor of the extended versions. The InputStream, Path, and filename-based constructor families each gain deprecation markers and updated `@see` tags pointing to the new signatures.

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · * @deprecated use {@link</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

```diff
@@ -149,7 +198,10 @@ public class EntryReader extends LineNumberReader implements Iterable<String>, I
    *     should define one group that contains the include file name.
    * @throws UnsupportedEncodingException if the charset encoding is not supported
    * @see #EntryReader(InputStream,String,String,String)
+   * @deprecated use {@link
+   *     #EntryReader(InputStream,String,String,boolean,String,String,String,String)}
    */
+  @Deprecated // 2026-01-05
   public @MustCallAlias EntryReader(
       @MustCallAlias InputStream in,
       String charsetName,
```

</details>

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · /**</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

```diff
@@ -348,6 +440,39 @@ public class EntryReader extends LineNumberReader implements Iterable<String>, I
 
   // Path constructors
 
+  /**
+   * Create an EntryReader.
+   *
+   * @param path initial file to read
+   * @param twoBlankLines true if entries are separated by two blank lines rather than one
+   * @param commentRegex regular expression that matches comments. Any text that matches
+   *     commentRegex is removed. A line that is entirely a comment is ignored.
+   * @param includeRegex regular expression that matches include directives. The expression should
+   *     define one group that contains the include file name.
+   * @param multilineCommentStart regular expression that matches the start of a multiline comment.
+   *     Any text that matches and follows after this regex is removed.
+   * @param multilineCommentEnd regular expression that matches the end of a multiline comment. Any
+   *     text that matches and precedes this regex is removed.
+   * @throws IOException if there is a problem reading the file
+   */
+  public EntryReader(
+      Path path,
+      boolean twoBlankLines,
+      @Nullable @Regex String commentRegex,
+      @Nullable @Regex(1) String includeRegex,
+      @Nullable @Regex String multilineCommentStart,
+      @Nullable @Regex String multilineCommentEnd)
+      throws IOException {
+    this(
+        FilesPlume.newFileReader(path),
+        path.toString(),
+        twoBlankLines,
+        commentRegex,
+        includeRegex,
+        multilineCommentStart,
+        multilineCommentEnd);
+  }
+
   /**
    * Create an EntryReader.
    *
```

</details>

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · * @deprecated use {@link #EntryReader(Path,boolean,String,S…</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

```diff
@@ -358,7 +483,9 @@ public class EntryReader extends LineNumberReader implements Iterable<String>, I
    * @param includeRegex regular expression that matches include directives. The expression should
    *     define one group that contains the include file name.
    * @throws IOException if there is a problem reading the file
+   * @deprecated use {@link #EntryReader(Path,boolean,String,String,String,String)}
    */
+  @Deprecated // 2026-01-05
   public EntryReader(
       Path path,
       boolean twoBlankLines,
```

</details>

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · * @see #EntryReader(Stream,String,String,boolean,String,Str…</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

```diff
@@ -404,8 +531,8 @@ public EntryReader(Path path) throws IOException {
    * @param path the file to read
    * @param charsetName the character set to use
    * @throws IOException if there is a problem reading the file
-   * @see #EntryReader(Stream,String,String,boolean,String,String)
-   * @deprecated use {@link #EntryReader(Path,boolean,String,String)}
+   * @see #EntryReader(InputStream,String,String,boolean,String,String,String,String)
+   * @deprecated use {@link #EntryReader(Path,boolean,String,String,String,String)}
    */
   @Deprecated // 2026-01-05
   public EntryReader(Path path, String charsetName) throws IOException {
```

</details>

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · * @param multilineCommentStart regular expression that matc…</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

```diff
@@ -423,8 +550,43 @@ public EntryReader(Path path, String charsetName) throws IOException {
    *     commentRegex is removed. A line that is entirely a comment is ignored.
    * @param includeRegex regular expression that matches include directives. The expression should
    *     define one group that contains the include file name.
+   * @param multilineCommentStart regular expression that matches the start of a multiline comment.
+   *     Any text that matches and follows after this regex is removed.
+   * @param multilineCommentEnd regular expression that matches the end of a multiline comment. Any
+   *     text that matches and precedes this regex is removed.
    * @throws IOException if there is a problem reading the file
    */
+  public EntryReader(
+      File file,
+      boolean twoBlankLines,
+      @Nullable @Regex String commentRegex,
+      @Nullable @Regex(1) String includeRegex,
+      @Nullable @Regex String multilineCommentStart,
+      @Nullable @Regex String multilineCommentEnd)
+      throws IOException {
+    this(
+        FilesPlume.newFileReader(file),
+        file.toString(),
+        twoBlankLines,
+        commentRegex,
+        includeRegex,
+        multilineCommentStart,
+        multilineCommentEnd);
+  }
+
+  /**
+   * Create an EntryReader.
+   *
+   * @param file initial file to read
+   * @param twoBlankLines true if entries are separated by two blank lines rather than one
+   * @param commentRegex regular expression that matches comments. Any text that matches
+   *     commentRegex is removed. A line that is entirely a comment is ignored.
+   * @param includeRegex regular expression that matches include directives. The expression should
+   *     define one group that contains the include file name.
+   * @throws IOException if there is a problem reading the file
+   * @deprecated use {@link #EntryReader(File,boolean,String,String,String,String)}
+   */
+  @Deprecated // 2026-01-05
   public EntryReader(
       File file,
       boolean twoBlankLines,
```

</details>

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · /**</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

```diff
@@ -480,6 +642,39 @@ public EntryReader(File file, String charsetName) throws IOException {
 
   // Filename constructors
 
+  /**
+   * Create a new EntryReader starting with the specified file.
+   *
+   * @param filename initial file to read
+   * @param twoBlankLines true if entries are separated by two blank lines rather than one
+   * @param commentRegex regular expression that matches comments. Any text that matches {@code
+   *     commentRegex} is removed. A line that is entirely a comment is ignored.
+   * @param includeRegex regular expression that matches include directives. The expression should
+   *     define one group that contains the include file name.
+   * @param multilineCommentStart regular expression that matches the start of a multiline comment.
+   *     Any text that matches and follows after this regex is removed.
+   * @param multilineCommentEnd regular expression that matches the end of a multiline comment. Any
+   *     text that matches and precedes this regex is removed.
+   * @throws IOException if there is a problem reading the file
+   * @see #EntryReader(File,boolean,String,String)
+   */
+  public EntryReader(
+      String filename,
+      boolean twoBlankLines,
+      @Nullable @Regex String commentRegex,
+      @Nullable @Regex(1) String includeRegex,
+      @Nullable @Regex String multilineCommentStart,
+      @Nullable @Regex String multilineCommentEnd)
+      throws IOException {
+    this(
+        new File(filename),
+        twoBlankLines,
+        commentRegex,
+        includeRegex,
+        multilineCommentStart,
+        multilineCommentEnd);
+  }
+
   /**
    * Create a new EntryReader starting with the specified file.
    *
```

</details>

<details open>
<summary><code>src/main/java/org/plumelib/util/EntryReader.java</code> · * @deprecated use {@link #EntryReader(String,boolean,String…</summary>

<!-- changetour:hunk file=src/main/java/org/plumelib/util/EntryReader.java level=2 baseBlob=068d5e2c1d4f31a103d38339673fa067f9c70adc -->

```diff
@@ -491,7 +686,9 @@ public EntryReader(File file, String charsetName) throws IOException {
    *     define one group that contains the include file name.
    * @throws IOException if there is a problem reading the file
    * @see #EntryReader(File,boolean,String,String)
+   * @deprecated use {@link #EntryReader(String,boolean,String,String,String,String)}
    */
+  @Deprecated // 2026-01-05
   public EntryReader(
       String filename,
       boolean twoBlankLines,
```

</details>

## Tests

A new `testMultilineComments` test exercises the feature, and the test class suppresses deprecation warnings since it continues to test the older constructor signatures.

<details open>
<summary><code>src/test/java/org/plumelib/util/EntryReaderTest.java</code> · "PMD.TooManyStaticImports"</summary>

<!-- changetour:hunk file=src/test/java/org/plumelib/util/EntryReaderTest.java level=2 baseBlob=190849c6ae711a39bca3f4ce00b47078f17e75b8 -->

```diff
@@ -25,7 +25,8 @@
 @SuppressWarnings({
   "nullness", // run-time errors are acceptable
   "initializedfields:contracts.postcondition", // @TempDir causes injection
-  "PMD.TooManyStaticImports"
+  "PMD.TooManyStaticImports",
+  "deprecation" // TODO
 })
 final class EntryReaderTest {
```

</details>

<details open>
<summary><code>src/test/java/org/plumelib/util/EntryReaderTest.java</code> · /** Test multiline comments */</summary>

<!-- changetour:hunk file=src/test/java/org/plumelib/util/EntryReaderTest.java level=2 baseBlob=190849c6ae711a39bca3f4ce00b47078f17e75b8 -->

```diff
@@ -447,4 +448,31 @@ void testEntryMetadata() throws IOException {
       assertEquals(2, entry.lineNumber); // line 2 after the leading blank line
     }
   }
+
+  /** Test multiline comments */
+  @Test
+  void testMultilineComments() throws IOException {
+    String content =
+        "pre\n```sh\ncode\n<!-- inside code should not be comment -->\n```\n<!--\nhidden1\nhidden2\n-->\npost\n";
+
+    try (EntryReader r =
+        new EntryReader(
+            new StringReader(content), "testfile.txt", false, null, null, "^<!--$", "^-->$")) {
+      assertEquals("pre", r.readLine());
+      assertEquals("```sh", r.readLine());
+      assertEquals("code", r.readLine());
+      // inside fenced code block; comment markers should be ignored
+      assertEquals("<!-- inside code should not be comment -->", r.readLine());
+      assertEquals("```", r.readLine());
+
+      // multiline comment block
+      assertEquals("", r.readLine()); // <!--
+      assertEquals("", r.readLine()); // hidden1
+      assertEquals("", r.readLine()); // hidden2
+      assertEquals("", r.readLine()); // -->
+
+      assertEquals("post", r.readLine());
+      assertNull(r.readLine());
+    }
+  }
 }
```

</details>

<!-- changetour:excluded-section -->

<!-- changetour:exclude file=.changetours/** -->
