# [FFmpeg 解析命令行参数流程](https://github.com/GeorgeCh2/blog/issues/3)

**FFmpeg 命令行基础语法：**

```shell
ffmpeg [global_options] {[input_file_options] -i input_file}...{[output_file_options] output_file}...
```

* global_options：全局参数
* input_file_options：输入文件相关参数
* output_file_options：输出文件相关参数

**如下为一个简单的 FFmpeg 命令，将 input.avi 视频文件转换为 640kbps 码率的 output.avi**

```
ffmpeg -i input.avi -b:v 640k output.avi
```



*当我们使用命令行来调用 FFmpeg 时，当命令行传入 FFmpeg 时，FFmpeg内部是如何识别这些命令并进行解析和赋值的呢？*

**首先简单看下流程图**：

![FFmpeg 解析命令行参数.png](https://upload-images.jianshu.io/upload_images/3911394-5f1624f8abfd8eeb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


总结起来，解析命令行的**大致流程**就是：

1. 跳过 “--xx xxx” 参数
2. “-xx xxx” 格式的默认参数存入全局参数数组或临时参数数组
3. “-noxx xxx”格式的参数，即默认值为“0”，将值存入全局参数数组或临时参数数组
4. 解析专属参数，并存入专属数组结构体（AVDictionary)
5. “-i xxx” 格式的输入文件路径参数，将临时参数数组的值、输入文件路径以及专属参数存入输入相关参数结构体，并清空临时参数数组
6. “xxx” 格式的输出文件路径参数，将临时参数数组的值、输出文件路径以及专属参数存入输出相关参数结构体，并清空临时参数数组



**参数解析后存放在哪？**

* 有关全局参数、输入参数、输出参数都存储到 `OptionParseContext *octx` 中

  ```c
  typedef struct OptionParseContext {
      // 全局命令分组
      OptionGroup global_opts;
  
      // 输入和输出的命令分组 (groups[0] 存储与输出文件相关参数，groups[1] 存储与输入文件相关参数)
      OptionGroupList *groups;
      int           nb_groups;
  
      /* 临时数组，存储输出、输入相关参数 */
      OptionGroup cur_group;
  } OptionParseContext;
  ```

* 专属参数会先存储到 AVDictionary

  ```c
  AVDictionary *codec_opts;
  AVDictionary *format_opts;
  AVDictionary *resample_opts;
  AVDictionary *sws_dict;
  AVDictionary *swr_opts;
  ```


**下面通过代码来分析 split_commandline 是如何解析命令行的：**


```c
/**
 * 解析命令行参数
 * @param octx:    用来将命令行中的参数通过分析分别存入此结构对应变量中
 * @param options: 默认参数的定义 (具体参见ffmpeg_opt.c文件中的 const OptionDef options[] 定义)
 * @param groups:  存储输入文件和输出文件相关参数
 * @param nb_groups: 2
 */
int split_commandline(OptionParseContext *octx, int argc, char *argv[],
                      const OptionDef *options,
                      const OptionGroupDef *groups, int nb_groups)
{
    int optindex = 1;
    int dashdash = -2;

    /* perform system-dependent conversions for arguments list */
    prepare_app_arguments(&argc, &argv);

    /* 基本的内存分配，OptionGroupDef 与 OptionParseContext 建立联系 */
    init_parse_context(octx, groups, nb_groups, data);
    av_log(NULL, AV_LOG_DEBUG, "Splitting the commandline.\n");

    /* 循环取出参数 */
    while (optindex < argc) {
        // 取参数队列
        const char *opt = argv[optindex++], *arg;
        const OptionDef *po;
        int ret;

        av_log(NULL, AV_LOG_DEBUG, "Reading option '%s' ...", opt);

        /* 跳过参数格式 --xx xxx */ 
        if (opt[0] == '-' && opt[1] == '-' && !opt[2]) {
            dashdash = optindex;
            continue;
        }
        /* 如果不是以 - 开头，另外一种情况，就是定义输出文件 */
        if (opt[0] != '-' || !opt[1] || dashdash+1 == optindex) {
            /** 
             * 将输出的路径、 octx->cur_group数组的值以及专属参数，存储到      			     
             * octx->groups[0]结构体中,并清空 octx->cur_group 数组的值
             */
            finish_group(octx, 0, opt);
            av_log(NULL, AV_LOG_DEBUG, " matched as %s.\n", groups[0].name);
            continue;
        }

        /* 排除以 -- 开头 & 输出文件定义，剩下的就是标准参数形式
           将参数的类型指针 +1，把 - 去掉 */
        opt++;

#define GET_ARG(arg)                                                           \
do {                                                                           \
    arg = argv[optindex++];                                                    \
    if (!arg) {                                                                \
        av_log(NULL, AV_LOG_ERROR, "Missing argument for option '%s'.\n", opt);\
        return AVERROR(EINVAL);                                                \
    }                                                                          \
} while (0)

        /* 参数格式是否为“-i xxx”，如果匹配，返回1 */
        if ((ret = match_group_separator(groups, nb_groups, opt)) >= 0) {
            // 获取参数对应的值
            GET_ARG(arg);
            /** 
             * 将输入的路径、octx->cur_group数组的值以及专属参数，存储到      			     
             * octx->groups[0]结构体中,并清空 octx->cur_group 数组的值
             */
            finish_group(octx, ret, arg);
            av_log(NULL, AV_LOG_DEBUG, " matched as %s with argument '%s'.\n",
                   groups[ret].name, arg);
            continue;
        }

        // 判断此 opt 是否为 options 中定义的参数
        po = find_option(options, opt);
        if (po->name) {
            if (po->flags & OPT_EXIT) {
                arg = argv[optindex++];
            } else if (po->flags & HAS_ARG) {
                /* 获取参数 */
                GET_ARG(arg);
            } else {
                // 允许参数后面的值缺失，直接设置为1
                arg = "1";
            }

            /**
             * 判断参数类型(opt->flags)，存入 octx->global_opts(全局参数) 或 
             * octx->cur_group(临时数组)
             */
            add_opt(octx, po, opt, arg);
            av_log(NULL, AV_LOG_DEBUG, " matched as option '%s' (%s) with "
                   "argument '%s'.\n", po->name, po->help, arg);
            continue;
        }

        if (argv[optindex]) {
            /**
             * 在 avcodec_options、avformat_options、avresample_options、   			 
             * swscale_options、swresample 中的 options 参数中不断查找，查找专属			  
             * 并存入 AVDictionary
             */
            ret = opt_default(NULL, opt, argv[optindex]);
            if (ret >= 0) {
                av_log(NULL, AV_LOG_DEBUG, " matched as AVOption '%s' with "
                       "argument '%s'.\n", opt, argv[optindex]);
                optindex++;
                continue;
            } else if (ret != AVERROR_OPTION_NOT_FOUND) {
                // 参数格式错误
                av_log(NULL, AV_LOG_ERROR, "Error parsing option '%s' "
                       "with argument '%s'.\n", opt, argv[optindex]);
                return ret;
            }
        }

        /* 处理 “-noxx" 格式参数 */
        if (opt[0] == 'n' && opt[1] == 'o' &&
            (po = find_option(options, opt + 2)) &&
            po->name && po->flags & OPT_BOOL) {
            // 存入默认值 ”0“
            add_opt(octx, po, opt, "0");
            av_log(NULL, AV_LOG_DEBUG, " matched as option '%s' (%s) with "
                   "argument 0.\n", po->name, po->help);
            continue;
        }

        // 没有匹配的参数，返回错误
        av_log(NULL, AV_LOG_ERROR, "Unrecognized option '%s'.\n", opt);
        return AVERROR_OPTION_NOT_FOUND;
    }

    // 循环结束后，临时参数及专属参数还存在值，没有存储到输入&输出相关参数数组中
    if (octx->cur_group.nb_opts || codec_opts || format_opts || resample_opts)
        av_log(NULL, AV_LOG_WARNING, "Trailing options were found on the "
               "commandline.\n");

    av_log(NULL, AV_LOG_DEBUG, "Finished splitting the commandline.\n");

    return 0;
}
```