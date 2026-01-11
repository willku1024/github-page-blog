+++
title = "Halide(2): gaussian模糊 + sobel边缘识别"
template = "page.html"
date = 2026-01-12T21:31:19
[taxonomies]
tags = ['Halide']
[extra]
+++

```cpp
#include "Halide.h"
#include "halide_image_io.h"
using namespace Halide;
using namespace Halide::Tools;

int main() {
    Var x("x"), y("y"), c("c");

    // 读输入图像
    Buffer<uint8_t> input = load_image("images/rgb.png");

    // 边界处理，避免越界
    Func clamped = BoundaryConditions::repeat_edge(input);

    // 转 float
    Func in_f("in_f");
    in_f(x, y, c) = cast<float>(clamped(x, y, c));

    // ---------- Stage 1: Gaussian blur ----------
    // 1D 核 [1, 2, 1]
    Buffer<float> gk(3);
    gk(0) = 1.0f; gk(1) = 2.0f; gk(2) = 1.0f;

    RDom rx(-1, 3), ry(-1, 3);

    Func blur_x("blur_x");
    blur_x(x, y, c) = 0.0f;
    blur_x(x, y, c) += in_f(x + rx, y, c) * gk(rx + 1);

    Func blur_y("blur_y");
    blur_y(x, y, c) = 0.0f;
    blur_y(x, y, c) += blur_x(x, y + ry, c) * gk(ry + 1);

    // 1-2-1 * 1-2-1 总权重 = 16
    Func gauss("gauss");
    gauss(x, y, c) = blur_y(x, y, c) / 16.0f;

    // ---------- Stage 2: Sobel on blurred image ----------
    // Sobel kernels
    Buffer<float> Kx(3, 3), Ky(3, 3);
    // Gx
    Kx(0,0) = -1; Kx(1,0) =  0; Kx(2,0) =  1;
    Kx(0,1) = -2; Kx(1,1) =  0; Kx(2,1) =  2;
    Kx(0,2) = -1; Kx(1,2) =  0; Kx(2,2) =  1;
    // Gy
    Ky(0,0) = -1; Ky(1,0) = -2; Ky(2,0) = -1;
    Ky(0,1) =  0; Ky(1,1) =  0; Ky(2,1) =  0;
    Ky(0,2) =  1; Ky(1,2) =  2; Ky(2,2) =  1;

    RDom k(0, 3, 0, 3);

    Func Gx("Gx"), Gy("Gy");
    Gx(x, y, c) = 0.0f;
    Gy(x, y, c) = 0.0f;

    // 在高斯模糊后的图上做 Sobel
    Gx(x, y, c) += gauss(x + k.x - 1, y + k.y - 1, c) * Kx(k.x, k.y);
    Gy(x, y, c) += gauss(x + k.x - 1, y + k.y - 1, c) * Ky(k.x, k.y);

    // 梯度幅值
    Func mag("mag");
    mag(x, y, c) = sqrt(Gx(x, y, c)*Gx(x, y, c) +
                        Gy(x, y, c)*Gy(x, y, c));

    Func output("output");
    output(x, y, c) = cast<uint8_t>(clamp(mag(x, y, c), 0.0f, 255.0f));

    // 可以先不写 schedule，直接 realize 一下
    Buffer<uint8_t> result =
        output.realize({input.width(), input.height(), input.channels()});

    save_image(result, "gauss_sobel.png");
    return 0;
}

```

---

<img src="/images/gaussian_sobel.png" style="width: 100%" alt>
