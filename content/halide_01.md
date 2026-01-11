+++
title = "Halide(1): blur(4种方式) & sobel & stencil 的实现"
template = "page.html"
date = 2026-01-11T22:31:19
[taxonomies]
tags = ['Halide']
[extra]
+++


```cpp
#include "Halide.h"
using namespace Halide;

// Support code for loading pngs.
#include "halide_image_io.h"
using namespace Halide::Tools;

int main() {

#if 0
     /*
      *   blur
      */

    // method 1: using boundary conditions
     {
        Var x("x"), y("y"), c("c");

        Buffer<uint8_t> input = load_image("images/rgb.ppm");

        // Upgrade it to 16-bit, so we can do math without it overflowing.
         Func input_16("input_16");
         input_16(x, y, c) = cast<float>(input(x, y, c));

        // Blur it horizontally:
        Func blur_x("blur_x");
        blur_x(x, y, c) = (input_16(x - 1, y, c) +
                            input_16(x, y, c) +
                           input_16(x + 1, y, c)) / 3.0f;

        // Blur it vertically:
        Func blur_y("blur_y");
        blur_y(x, y, c) = (blur_x(x, y - 1, c) +
                            blur_x(x, y, c) +
                           blur_x(x, y + 1, c)) / 3.0f;


        // Convert back to 8-bit.
        Func output("output");


        output(x, y, c) = cast<uint8_t>(blur_y(x, y, c));

        // !!! use clipped area, instead of clamp()
        // 越界了，不能这么用：   Buffer<uint8_t> result = output.realize({input.width(), input.height(), 3});
        Buffer<uint8_t> result(input.width() - 2, input.height() - 2, 3);
        result.set_min(1, 1);
        output.realize(result);

        save_image(result, "blur_1.ppm");
    }

    // method 2: using clamped function
    {
        Var x("x"), y("y"), c("c");

        Buffer<uint8_t> input = load_image("images/rgb.ppm");

        Func clamped;
        Expr x_clamped = clamp(x, 0, input.width() - 1);
        Expr y_clamped = clamp(y, 0, input.height() - 1);
        clamped(x, y, c) = input(x_clamped, y_clamped, c);

        // Upgrade it to 16-bit, so we can do math without it overflowing.
        Func input_16("input_16");
        input_16(x, y, c) = cast<float>(clamped(x, y, c));

        // Blur it horizontally:
        Func blur_x("blur_x");
        blur_x(x, y, c) = (input_16(x - 1, y, c) +
                            input_16(x, y, c) +
                           input_16(x + 1, y, c)) / 3.0f;

        // Blur it vertically:
        Func blur_y("blur_y");
        blur_y(x, y, c) = (blur_x(x, y - 1, c) +
                            blur_x(x, y, c) +
                           blur_x(x, y + 1, c)) / 3.0f;

        // Convert back to 8-bit.
        Func output("output");
        output(x, y, c) = cast<uint8_t>(blur_y(x, y, c));

        Buffer<uint8_t> result = output.realize({input.width(), input.height(), 3});
        save_image(result, "blur_2.ppm");

    }

    // method 3: using Halide's built-in boundary conditions
    {
        Var x("x"), y("y"), c("c");

        Buffer<uint8_t> input = load_image("images/rgb.ppm");

        Func clamped = BoundaryConditions::repeat_edge(input);

        // Upgrade it to 16-bit, so we can do math without it overflowing.
        Func input_16("input_16");
        input_16(x, y, c) = cast<float>(clamped(x, y, c));

        // Blur it horizontally:
        Func blur_x("blur_x");
        blur_x(x, y, c) = (input_16(x - 1, y, c) +
                            input_16(x, y, c) +
                           input_16(x + 1, y, c)) / 3.0f;

        // Blur it vertically:
        Func blur_y("blur_y");
        blur_y(x, y, c) = (blur_x(x, y - 1, c) +
                            blur_x(x, y, c) +
                           blur_x(x, y + 1, c)) / 3.0f;

        // Convert back to 8-bit.
        Func output("output");
        output(x, y, c) = cast<uint8_t>(blur_y(x, y, c));

        Buffer<uint8_t> result = output.realize({input.width(), input.height(), 3});
        save_image(result, "blur_3.ppm");
    }


    // method 4: use RDom for kernel offset
    {
         Var x("x"), y("y"), c("c");

         Buffer<uint8_t> input = load_image("images/rgb.ppm");

         // 边界处理
         Func clamped = BoundaryConditions::repeat_edge(input);

         // 提升到 float
         Func input_f("input_f");
         input_f(x, y, c) = cast<float>(clamped(x, y, c));

         // 3×3 box blur，用 RDom 描述卷积核偏移
         RDom k(-1, 3,   // k.x = -1,0,1
                -1, 3);  // k.y = -1,0,1

         Func blur("blur");
         blur(x, y, c) = 0.0f;
         blur(x, y, c) += input_f(x + k.x, y + k.y, c);

         // 取平均 & 转回 8-bit
         Func output("output");
         output(x, y, c) = cast<uint8_t>(blur(x, y, c) / 9.0f);

         Buffer<uint8_t> result =
             output.realize({input.width(), input.height(), input.channels()});

         save_image(result, "blur_4.ppm");
    }

#endif


#if 0
  /*
   *   sobel
  */

  // method 1:
 {

       Func gx, gy, mag[3];
       Var x("x"), y("y"), c("c");
       Buffer<uint8_t> input = load_image("images/rgb.ppm");
       Func clamped = BoundaryConditions::repeat_edge(input);

       Buffer<float> Kx(3, 3);
       Kx(0,0) = -1;  Kx(1,0) = 0;  Kx(2,0) = 1;
       Kx(0,1) = -2;  Kx(1,1) = 0;  Kx(2,1) = 2;
       Kx(0,2) = -1;  Kx(1,2) = 0;  Kx(2,2) = 1;

       Buffer<float> Ky(3, 3);
       Ky(0,0) = -1; Ky(1,0) = -2; Ky(2,0) = -1;
       Ky(0,1) =  0; Ky(1,1) =  0; Ky(2,1) =  0;
       Ky(0,2) =  1; Ky(1,2) =  2; Ky(2,2) =  1;


       RDom r(0, 3, 0, 3);
       gx(x, y, c) = 0.0f;
       gx(x, y, c) += clamped(x + r.x - 1, y + r.y - 1, c) * Kx(r.x, r.y);


       gy(x, y, c) = 0.0f;
       gy(x, y, c) += clamped(x + r.x - 1, y + r.y - 1, c) * Ky(r.x, r.y);


       // L2
       mag[0](x , y, c) = sqrt(gx(x , y, c) * gx(x , y, c)
                                     + gy(x , y, c) * gy(x , y, c));

       // L1
       mag[1](x , y, c) = abs(gx(x , y, c) * gx(x , y, c))
                              + abs(gy(x , y, c) * gy(x , y, c));

       // L2 no sqrt
       mag[2](x , y, c) = (gx(x , y, c) * gx(x , y, c))
                              + (gy(x , y, c) * gy(x , y, c));


       for (int i = 0; i < 3; i++) {
         Func output("output");
         output(x, y, c) = cast<uint8_t>(clamp(mag[i](x, y, c), 0, 255));
         Buffer<uint8_t> result = output.realize({input.width(), input.height(), input.channels()});

         char filename[50] = {0};
         snprintf(filename, 49, "sobel_1_%d.ppm", i);
         save_image(result, filename);
       }

 }
#endif

#if 1
       {
         Var x, y, c;
         Buffer<uint8_t> input = load_image("images/rgb.ppm");

         Func clamped = BoundaryConditions::repeat_edge(input);

         Func stencil("stencil");

         Expr sum =
             clamped(x,     y,     c) +
             clamped(x + 1, y,     c) +
             clamped(x - 1, y,     c) +
             clamped(x,     y + 1, c) +
             clamped(x,     y - 1, c);

         stencil(x, y, c) = cast<uint8_t>(sum / 5);

         Buffer<uint8_t> result =
             stencil.realize({input.width(), input.height(), input.channels()});

         save_image(result, "simple_stencil.ppm");
     }
#endif

    return 0;
}

```
