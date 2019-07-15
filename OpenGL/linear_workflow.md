# OpenGL - Linear workflow

大雑把な前提条件

1. フェッチしたテクスチャの色は必ず線形色空間にある
2. 光源計算をはじめ計算は線形色空間で行われる
3. 最終的にディスプレイに出力される直前に非線形色空間に戻す



以下では、

- 線形色空間を`Linear`
- 非線形色空間を`sRGB`

と呼称する。



## 1. フェッチしたテクスチャの色は必ず線形色空間にある

1. ロードするテクスチャをすべて`Linear`とみなす

   やれなくもないけど･･･。

   

2. ハードウェアに丸投げする

   1. `sRGB`  イメージ

      特に事情がなければ世の中の大抵のイメージはこちらに該当。

      ```c++
      glTexImage2D(GL_TEXTURE_2D, 0, GL_SRGB8, w, h, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
      ```

      または

      ```c++
      glTexImage2D(GL_TEXTURE_2D, 0, GL_SRGB8_ALPHA8, w, h, 0, GL_RGBA, GL_UNSIGNED_BYTE, data);
      ```

      これで生成されたテクスチャはハードウェアが`sRGB`→`Linear`に変換してくれる。

      cf. [glTexImage2D](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glTexImage2D.xhtml)

      

   2. `Linear` イメージ

      色とは無関係の法線マップや高さマップなど。

      ```c++
      glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, w, h, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
      ```

      または

      ```c++
      glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, w, h, 0, GL_RGBA, GL_UNSIGNED_BYTE, data);
      ```

      当然ハードウェアは何もしない。

      

   3. `HDRI` イメージ

      そもそも`Linear`。

   

   これで以降の入力はすべて`Linear`が保証される。

   

3. フラグメントシェーダーで行う

   入力が`sRGB`なら`sRGB`→`Linear`に変換してから各種計算を行う。

   常に入力として受け取るテクスチャの色空間を把握しておく必要がある。

   ※煩わしいがやりたいことは何でもできる。



## 2. 光源計算をはじめ計算は線形色空間で行われる

入力はすべて`Linear`であることが保証されているとして、中間`FBO`もすべて`Linear`なら間違いは起こらない。

1つでも`sRGB`の`FBO`が混じっている場合は注意。



**確認方法**

下記は0番のカラーバッファだけが有効な`FBO`

```c++
GLuint fbo = 0;
glGenFramebuffers(1, &fbo);
glBindFramebuffer(GL_FRAMEBUFFER, fbo);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, texture, 0);
```

次のように確認を行う。

```c++
glBindFramebuffer(GL_FRAMEBUFFER, fbo);
// ...
GLint encoding = 0;
glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_FRAMEBUFFER_ATTACHMENT_COLOR_ENCODING, &encoding);
if(encoding == GL_LINEAR)
{
    // (1) Linear
}
else if(encoding == GL_SRGB)
{
    // (2) sRGB
}
else
{
    // (3) If nothing is attached to the fbo.
}
```

(3) は、カラーバッファがアタッチされていない場合に該当。

※故にデバッグ表示に三項演算子などを利用すると混乱するので注意



## 3. 最終的にディスプレイに出力される直前に非線形色空間に戻す

`Linear`→`sRGB`に変換。

1. ハードウェアに丸投げする

   ```c++
   glEnable(GL_FRAMEBUFFER_SRGB);
   render_xxx_from_linear_to_srgb();  // to_srgb: backbuffer or sRGB-FBO.
   glDisable(GL_FRAMEBUFFER_SRGB);
   ```

   `Linear`な`FBO`には影響がないらしいので、全体が単純なら初期化時に一度設定しておくだけでも良い。

   **※ ただし、`GL_FRAMEBUFFER_SRGB`が正常に機能しないハードウェアもあるらしい。**

   

2. フラグメントシェーダーで行う

   フラグメントシェーダーの最終出力を`Linear`→`sRGB`に変換する。

   バックバッファへ出力するシェーダーが複数ある場合はすべてに必要となる。

   故に大したことをしなくても可能ならポストプロセス処理を最低１つ設けると楽になる。



## #. デフォルト・フレームバッファ

**2.** で確認した方法をデフォルト・フレームバッファについても行ってみる。

調査環境は Windows10 / NVIDIA GeForce GTX 760 (v430.86)  。

```c++
const std::unordered_map<GLenum, std::string> default_buffer =
{
    {GL_FRONT_LEFT, "GL_FRONT_LEFT"},
    {GL_FRONT_RIGHT, "GL_FRONT_RIGHT"},
    {GL_BACK_LEFT, "GL_BACK_LEFT"},
    {GL_BACK_RIGHT, "GL_BACK_RIGHT"}
};

const std::unordered_map<GLint, std::string> color_encoding =
{
    {GL_LINEAR, "GL_LINEAR"},
    {GL_SRGB, "GL_SRGB"}
};

glBindFramebuffer(GL_FRAMEBUFFER, 0);

for(auto& pair : default_buffer)
{
    GLint encoding = 0;
    glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, pair.first, GL_FRAMEBUFFER_ATTACHMENT_COLOR_ENCODING, &encoding);

    auto it = color_encoding.find(encoding);
    std::string name = (it != color_encoding.cend())? it->second : "Unknown";
    std::cout << "default_buffer: " << pair.second << " encoding: " << name << std::endl;
}
```

他の影響がないように調査前に必ず`glBindFramebuffer(GL_FRAMEBUFFER, 0);`を忘れずに。

```shell
default_buffer: GL_FRONT_LEFT encoding: GL_LINEAR
default_buffer: GL_FRONT_RIGHT encoding: GL_LINEAR
default_buffer: GL_BACK_LEFT encoding: GL_LINEAR
default_buffer: GL_BACK_RIGHT encoding: GL_LINEAR
```

通常、デフォルト・フレームバッファは`sRGB`を想定するので、この結果は明らかに問題がある。

しかし実際の環境で`GL_FRAMEBUFFER_SRGB`を有効にすると`Linear`→`sRGB`への変換が期待どおり行われる。



問題提起されてから随分と放置されているようなので、他環境(ハードウェア・OS)も考慮するなら`GL_FRAMEBUFFER_SRGB`は選択肢から外す必要がありそうだ。



### 参考:

1. [LearnOpenGL - Gamma Correction](https://learnopengl.com/Advanced-Lighting/Gamma-Correction)

2. [March 2015 OpenGL drivers status and FB sRGB conversions](https://www.g-truc.net/post-0720.html)

3. [GL_FRAMEBUFFER_SRGB functions incorrectly](https://devtalk.nvidia.com/default/topic/776591/opengl/gl_framebuffer_srgb-functions-incorrectly/)

   

   

