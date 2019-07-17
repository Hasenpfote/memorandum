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
      
      `OpenEXR`などの半精度浮動小数点数の場合、
      
      ```c++
      glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB16F, w, h, 0, GL_RGB, GL_HALF_FLOAT, data);
      ```
      
      または、
      
      ```c++
      glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA16F, w, h, 0, GL_RGBA, GL_HALF_FLOAT, data);
      ```
      
      
      
      `OpenEXR`や`Radiance`などの単精度浮動小数点数の場合、
      
      ```c++
      glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB32F, w, h, 0, GL_RGB, GL_FLOAT, data);
      ```
      
      または、
      
      ```c++
      glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA32F, w, h, 0, GL_RGBA, GL_FLOAT, data);
      ```
      
      
      
      cf. [OpenEXR](https://www.openexr.com/)
      
      cf. [Radiance](https://www.radiance-online.org/)
      
      cf. [Radiance File Formats](https://floyd.lbl.gov/radiance/refer/filefmts.pdf)
      
      cf. [The RADIANCE Picture File Format](https://floyd.lbl.gov/radiance/refer/Notes/picture_format.html)

   

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

cf. [glGetFramebufferAttachmentParameteriv](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glGetFramebufferAttachmentParameter.xhtml)



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
static void dump_framebuffer_info(GLenum attachment)
{
    static const std::unordered_map<GLint, std::string> _object_type =
    {
        {GL_NONE, "GL_NONE"},
        {GL_FRAMEBUFFER_DEFAULT, "GL_FRAMEBUFFER_DEFAULT"},
        {GL_TEXTURE, "GL_TEXTURE"},
        {GL_RENDERBUFFER, "GL_RENDERBUFFER"}
    };
    static const std::unordered_map<GLint, std::string> _component_type =
    {
        {GL_FLOAT, "GL_FLOAT"},
        {GL_INT, "GL_INT"},
        {GL_UNSIGNED_INT, "GL_UNSIGNED_INT"},
        {GL_SIGNED_NORMALIZED, "GL_SIGNED_NORMALIZED"},
        {GL_UNSIGNED_NORMALIZED, "GL_UNSIGNED_NORMALIZED"},
        {GL_NONE, "GL_NONE"}
    };
    static const std::unordered_map<GLint, std::string> _color_encoding =
    {
        {GL_LINEAR, "GL_LINEAR"},
        {GL_SRGB, "GL_SRGB"}
    };
    static const std::unordered_map<GLint, std::string> _layered =
    {
        {GL_TRUE, "GL_TRUE"},
        {GL_FALSE, "GL_FALSE"}
    };

    auto stringify = [](const auto& umap, GLint key)
    {
        auto it = umap.find(key);
        return (it != umap.cend()) ? it->second : "Unknown";
    };

    GLint object_type = 0;
    glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_OBJECT_TYPE, &object_type);
    std::cout << "object_type: " << stringify(_object_type, object_type) << std::endl;

    if(object_type == GL_NONE)
    {
    }
    else
    {
        GLint r_size = 0;
        glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_RED_SIZE, &r_size);
        GLint g_size = 0;
        glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_GREEN_SIZE, &g_size);
        GLint b_size = 0;
        glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_BLUE_SIZE, &b_size);
        GLint a_size = 0;
        glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_ALPHA_SIZE, &a_size);
        GLint d_size = 0;
        glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_DEPTH_SIZE, &d_size);
        GLint s_size = 0;
        glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_STENCIL_SIZE, &s_size);
        std::cout << "r_size: " << r_size << " g_size: " << g_size << " b_size: " << b_size << " a_size: " << a_size << " d_size: " << d_size << " s_size: " << s_size << std::endl;

        GLint component_type = 0;
        glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_COMPONENT_TYPE, &component_type);
        std::cout << "component_type: " << stringify(_component_type, component_type) << std::endl;

        GLint color_encoding = 0;
        glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_COLOR_ENCODING, &color_encoding);
        std::cout << "color_encoding: " << stringify(_color_encoding, color_encoding) << std::endl;

        if(object_type == GL_RENDERBUFFER)
        {
            GLint object_name = 0;
            glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_OBJECT_NAME, &object_name);
            std::cout << "object_name: " << object_name << std::endl;
        }
        else if(object_type == GL_TEXTURE)
        {
            GLint object_name = 0;
            glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_OBJECT_NAME, &object_name);
            std::cout << "object_name: " << object_name << std::endl;

            GLint texture_level = 0;
            glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_TEXTURE_LEVEL, &texture_level);
            std::cout << "texture_level: " << texture_level << std::endl;

            GLint texture_cube_map_face = 0;
            glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_TEXTURE_CUBE_MAP_FACE, &texture_cube_map_face);
            std::cout << "texture_cube_map_face: " << texture_cube_map_face << std::endl;

            GLint layered = 0;
            glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_LAYERED, &layered);
            std::cout << "layered: " << stringify(_layered, layered) << std::endl;

            GLint texture_layer = 0;
            glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER, attachment, GL_FRAMEBUFFER_ATTACHMENT_TEXTURE_LAYER, &texture_layer);
            std::cout << "texture_layer: " << texture_layer << std::endl;
        }
        else if(object_type == GL_FRAMEBUFFER_DEFAULT)
        {
        }
    }
}
```

他の影響がないように調査前に必ず`glBindFramebuffer(GL_FRAMEBUFFER, 0);`を忘れずに。

```c++
static void dump_default_framebuffer_info()
{
    const std::unordered_map<GLenum, std::string> _default_framebuffer =
    {
        {GL_FRONT_LEFT, "GL_FRONT_LEFT"},
        {GL_FRONT_RIGHT, "GL_FRONT_RIGHT"},
        {GL_BACK_LEFT, "GL_BACK_LEFT"},
        {GL_BACK_RIGHT, "GL_BACK_RIGHT"}
    };

    GLint prev_fbo = 0;
    glGetIntegerv(GL_FRAMEBUFFER_BINDING, &prev_fbo);
    glBindFramebuffer(GL_FRAMEBUFFER, 0);

    for(auto& pair : _default_framebuffer)
    {
        std::cout << "*** default_framebuffer: " << pair.second << std::endl;
        dump_framebuffer_info(pair.first);
    }

    glBindFramebuffer(GL_FRAMEBUFFER, prev_fbo);
}
```

実行すると、

```shell
*** default_framebuffer: GL_FRONT_LEFT
object_type: GL_FRAMEBUFFER_DEFAULT
r_size: 8 g_size: 8 b_size: 8 a_size: 8 d_size: 0 s_size: 0
component_type: GL_UNSIGNED_NORMALIZED
color_encoding: GL_LINEAR

*** default_framebuffer: GL_FRONT_RIGHT
object_type: GL_FRAMEBUFFER_DEFAULT
r_size: 0 g_size: 0 b_size: 0 a_size: 0 d_size: 0 s_size: 0
component_type: GL_UNSIGNED_NORMALIZED
color_encoding: GL_LINEAR

*** default_framebuffer: GL_BACK_LEFT
object_type: GL_FRAMEBUFFER_DEFAULT
r_size: 8 g_size: 8 b_size: 8 a_size: 8 d_size: 0 s_size: 0
component_type: GL_UNSIGNED_NORMALIZED
color_encoding: GL_LINEAR

*** default_framebuffer: GL_BACK_RIGHT
object_type: GL_FRAMEBUFFER_DEFAULT
r_size: 0 g_size: 0 b_size: 0 a_size: 0 d_size: 0 s_size: 0
component_type: GL_UNSIGNED_NORMALIZED
color_encoding: GL_LINEAR
```

通常、デフォルトフレームバッファは`sRGB`を想定しそうなものだが、どうもそうではないようだ。

実環境で`GL_FRAMEBUFFER_SRGB`を有効にすると`Linear`→`sRGB`への変換が期待どおり行われる。

一部の他社ハードウェアやOSでは一致するかは不明。

※仕様(NVIDIA)に一部の他社ハードウェアが追い付いていない？



### 参考:

1. [LearnOpenGL - Gamma Correction](https://learnopengl.com/Advanced-Lighting/Gamma-Correction)

2. [March 2015 OpenGL drivers status and FB sRGB conversions](https://www.g-truc.net/post-0720.html)

3. [GL_FRAMEBUFFER_SRGB functions incorrectly](https://devtalk.nvidia.com/default/topic/776591/opengl/gl_framebuffer_srgb-functions-incorrectly/)

   

   

