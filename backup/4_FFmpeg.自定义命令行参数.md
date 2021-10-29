# [FFmpeg 自定义命令行参数](https://github.com/GeorgeCh2/blog/issues/4)

我们在使用 FFmpeg 的时候，会发现 FFmpeg 有些库的性能并不是特别的好，可能就想要使用其他性能更好的第三方SDK 或 自己开发的SDK来替换。这时可能 FFmpeg 的默认命令行参数并不能我们的需求，就需要自定义命令行参数。那么如何来自定义命令行参数达到我们的需求呢？

此次我们在 FFmpeg 中增加了 libyuv 的图像缩放算法，那么就可以在 swscale_options[]（FFmpeg 内置 swscale 过滤器相关参数数组） 中添加一个 “scale_method” 用来选择图像缩放算法。

**添加参数前，需要先做一些前置动作**：

* 定义 scale_method 参数
  在 swscale_internal.h 的 SwsContext 结构体中添加一个属性 `int scale_method;`  
  FFmpeg 接收参数后进行解析，并将用户传入的值赋值给对应结构体内的属性。
* 定义参数选项  
  在 swscale.h 中定义：
  ```c
  // libyuv 的 scale 方法
  #define SWS_SCALE             1
  #define LIB_YUV_NONE          2
  #define LIB_YUV_BILINEAR      3
  #define LIB_YUV_BOX           4
  ```

**添加前面的定义后，就可以添加自定义参数了**：
​    在 libswscale/options.c 文件中的 swscale_options[] 数组中添加：
```c
{ "scale_method",    "choice swscale or libyuv to scale", OFFSET(scale_method), AV_OPT_TYPE_FLAGS,  { .i64 = SWS_SCALE          },  1,       2,              VE, "scale_method"},
    { "swscale",         "use swscale to scale",              0,                    AV_OPT_TYPE_CONST,  { .i64 = SWS_SCALE          },  INT_MIN, INT_MAX,        VE, "scale_method"},
    { "libyuv_none",     "use libyuv KFilterNone scale",      0,                    AV_OPT_TYPE_CONST,  { .i64 = LIB_YUV_NONE       },  INT_MIN, INT_MAX,        VE, "scale_method" },
    { "libyuv_bilinear", "use libyuv KFilterBilinear scale",  0,                    AV_OPT_TYPE_CONST,  { .i64 = LIB_YUV_BILINEAR   },  INT_MIN, INT_MAX,        VE, "scale_method" },
    { "libyuv_box",      "use libyuv KFilterBox scale",       0,                    AV_OPT_TYPE_CONST,  { .i64 = LIB_YUV_BOX        },  INT_MIN, INT_MAX,        VE, "scale_method" },

```

对应各个变量的定义如下：
{名称， 简介，相对结构体首部地址的偏移量，选项的类型，选项的默认值，选项的最小值，选项的最大值，标记，所属的逻辑单元（可为空）}
* 添加完自定义参数后就可以在需要更改的地方获取参数，然后进行相应的逻辑更改
  ```c
  int scale_method = sws->scale_method;
      if (scale_method == LIB_YUV_BILINEAR || scale_method == LIB_YUV_BOX || scale_method == LIB_YUV_NONE) {
          enum FilterMode mode =  kFilterNone;
          switch (scale_method)
          {
              case LIB_YUV_BILINEAR :
                  mode = kFilterBilinear;
                  break;
              case LIB_YUV_BOX :
                  mode = kFilterBox;
                  break;
              default:
                  break;
          }
          // 使用 libyuv 的 scale 方法
          return I420Scale(in[0], in_stride[0], in[1], in_stride[1], in[2], in_stride[2], 
                  cur_pic->width, cur_pic->height, 
                  out[0], out_stride[0], out[1], out_stride[1], out[2], out_stride[2], 
                  out_buf->width, out_buf->height, mode);
      } 
      else
      {
          // 使用 sws_scale
          return sws_scale(sws, in, in_stride, y/mul, h,
                            out,out_stride);
      }
  ```


添加完自定义参数后，就可以通过传入 `"-scale_method","libyuv_none"` 来选择 KFilterNone 图像缩放算法.
**其他的参数也可参照着去更改**：
* 首先定位需要修改的地方
* 然后根据业务找到参数定义的位置（默认参数或 avcodec_options、avformat_options、avresample_options等专属参数）
* 在对应参数数组中添加自定义参数&对应结构体的属性
* 获取结构体属性进行相应逻辑更改。