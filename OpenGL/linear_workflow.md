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

      これで生成されたテクスチャはフェッチ時にハードウェアが`sRGB`→`Linear`に変換してくれる。

      

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



## 3. 最終的にディスプレイに出力される直前に非線形色空間に戻す

`Linear`→`sRGB`に変換。

1. ハードウェアに丸投げする

   ```c++
   glEnable(GL_FRAMEBUFFER_SRGB);
   render_xxx_from_linear_to_srgb();  // to_srgb: backbuffer or sRGB-FBO.
   glDisable(GL_FRAMEBUFFER_SRGB);
   ```

   `Linear`な`FBO`には影響がないらしいので、全体が単純なら初期化時に一度設定しておくだけでも良い。

   

2. フラグメントシェーダーで行う

   フラグメントシェーダーの最終出力を`Linear`→`sRGB`に変換する。

   バックバッファへ出力するシェーダーが複数ある場合はすべてに必要となる。

   ので、大したことをしなくても可能ならポストプロセス処理を最低１つ設けると楽になる。

**※ 1 については、`GL_FRAMEBUFFER_SRGB` が機能しないハードウェアもあるらしい。**



### 参考:

1. [LearnOpenGL - Gamma Correction](https://learnopengl.com/Advanced-Lighting/Gamma-Correction)

