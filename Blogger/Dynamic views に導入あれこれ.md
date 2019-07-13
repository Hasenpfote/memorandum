# Dynamic views に導入あれこれ  



## [google/code-prettify](https://github.com/google/code-prettify)

1. `<head>` ～`</head>`に下記を追加

   ```html
   <script src='https://cdn.rawgit.com/google/code-prettify/master/loader/run_prettify.js?skin=Default'/>
   ```

2. `<pre>`または`<code>を`利用

   ```html
   <pre class="prettyprint lang-c">
   int main()
   {
       return 0;
   }
   </pre>
   ```

3. 各記事にスクリプトを埋め込む

   ```html
   <script>PR.prettyPrint()</script>
   ```

   

## [highlight.js](https://highlightjs.org/)

1. `<head>` ～`</head>`に下記を追加

   ```html
   <link href='http://cdnjs.cloudflare.com/ajax/libs/highlight.js/9.10.0/styles/default.min.css' rel='stylesheet'/>
   <script src='http://cdnjs.cloudflare.com/ajax/libs/highlight.js/9.10.0/highlight.min.js'/>
   ```

2. `<pre>`と`<code>`を利用

   ```html
   <pre><code class="cpp">
   int main()
   {
       return 0;
   }
   </code></pre>
   ```

3. 各記事にスクリプトを埋め込む

   ```html
   <script>
       // script initialization for 'pre' tags.
       $(document).ready(function() {
           $('pre').each(function(i, block) {
               hljs.highlightBlock(block);
           });
       });
   </script>
   ```



## [mathjax](https://www.mathjax.org/)

1. `<head>` ～`</head>`に下記を追加

   ```html
   <script src='http://cdn.mathjax.org/mathjax/latest/MathJax.js' type='text/javascript'>    
       MathJax.Hub.Config({
           HTML: ["input/TeX","output/HTML-CSS"],
           TeX: { extensions: ["AMSmath.js","AMSsymbols.js"], 
                  equationNumbers: { autoNumber: "AMS" } },
           extensions: ["tex2jax.js"],
           jax: ["input/TeX","output/HTML-CSS"],
           tex2jax: { inlineMath: [ ['$','$'], ["\\(","\\)"] ],
                      displayMath: [ ['$$','$$'], ["\\[","\\]"] ],
                      processEscapes: true },
           "HTML-CSS": { availableFonts: ["TeX"],
                         linebreaks: { automatic: true } }
       });
   </script>
   ```

2. `tex2jax`に利用できる表記があるのでそれに基づき数式を書く

   ```html
   $$
   e^{i\pi}=-1
   $$
   ```

   

3. 各記事にスクリプトを埋め込む

   ```html
   <script type="text/javascript">MathJax.Hub.Queue(["Typeset",MathJax.Hub]);</script>
   ```



## StackEdit → Blogger

1. 言語のハイライト

   `hilight.js`で対応

2. 数式の表示

   `MathJax` で対応。

   ただし、`StackEdit` は `Katex` を利用しているため二重表示されるなど問題がある。

   これを回避するには、`Blogger` にアップロードするときに

   [Menu] → [File properties]  の [YAML properties] に下記を設定する。 

   ```yaml
   extensions:
     katex:
       enabled: false
   status: draft
   ```

   ※ アップロード時以外は有効にしないと数式がレンダリングされないので注意。

3. その他の表示調整

   カスタム CSS で対応。
   
   タグの設定が干渉しないように注意する必要がある。
   
   個人的には`<code>`と`<table>`は対応しておきたかったので[こちら](for_blogger.css)のような設定を作った。
   
   `[テーマ]` → `[カスタマイズ]`  → `[上級者向け]` → `[CSSを追加]`に `css` ファイルの内容をコピペ。 



実例は[こちら](https://hasenpfote36-2.blogspot.com/2019/07/stackedit-to-blogger-dynamic-views_13.html)。



## ...







